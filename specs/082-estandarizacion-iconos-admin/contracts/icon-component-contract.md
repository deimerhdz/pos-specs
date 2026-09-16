# Contrato: Componente de Ícono Reutilizable (panel de administración)

**Spec**: [../spec.md](../spec.md) | **Plan**: [../plan.md](../plan.md) | **Decisiones**: [../research.md](../research.md) (D1–D3)

Este documento define el contrato público del componente Angular nuevo que reemplaza, dentro del
panel de administración, tanto al componente SVG artesanal (`app-icon`) como a los emoji usados como
ícono. No reemplaza ni modifica `app-icon` (research.md, D2) — es un componente independiente, nuevo.

## Selector y ubicación

- Selector distinto al existente `app-icon` (a definir en Fase 2/implementación, p. ej. `app-mi-icon`
  o equivalente), para no colisionar ni sugerir intercambiabilidad con el componente artesanal que
  sigue sirviendo al flujo público.
- Ubicado junto al resto de utilidades compartidas del panel de administración.
- Standalone, consistente con el resto de la base de código (`standalone: true`, sin `NgModule`).

## Inputs

| Input | Tipo | Requerido | Descripción |
|---|---|---|---|
| `name` | `string` | Sí | Nombre semántico del ícono (ver `data-model.md` para el catálogo nombre → ligadura de Material Icons). El componente resuelve internamente el nombre semántico contra el nombre real de la ligadura de Material Icons — las plantillas que lo consumen nunca escriben directamente el nombre de la ligadura de Google. |
| `ariaLabel` | `string` | No | Descripción accesible equivalente al significado del ícono (FR-002, FR-011). Cuando se provee, el ícono se anuncia a lectores de pantalla; cuando se omite, el ícono se trata como decorativo. |
| `size` | `number` (píxeles) | No (por defecto `20`) | Tamaño del ícono. A diferencia de `app-icon` (SVG dimensionado por `w-*`/`h-*` del contenedor), este componente renderiza una ligadura de fuente dimensionada por `font-size`; `size` traduce el ancho/alto en píxeles del ícono reemplazado a un `font-size` equivalente, para no perder fidelidad visual al migrar. |

## Comportamiento de renderizado

- Si `ariaLabel` está presente: el elemento raíz se renderiza con `role="img"` y
  `aria-label="{{ ariaLabel }}"`.
- Si `ariaLabel` está ausente: el elemento raíz se renderiza con `aria-hidden="true"` (decorativo —
  se asume que el texto visible adyacente o el elemento interactivo contenedor ya expone su propio
  nombre accesible).
- El color se hereda del elemento padre (`color: inherit`, mismo patrón que `currentColor` usaba en
  el SVG) — ninguna pantalla necesita cambiar su forma de controlar el color al migrar. El tamaño,
  en cambio, se fija explícitamente con el input `size` (ver tabla de Inputs) en vez de heredarse,
  porque una ligadura de fuente no se redimensiona sola al ancho/alto de un contenedor como sí lo
  hacía el SVG.
- Si `name` no corresponde a ninguna entrada del catálogo (`data-model.md`), el componente renderiza
  un ícono neutro de reserva (p. ej. `help_outline`) en vez de quedar vacío o roto — mismo principio
  de "nunca vacío" que ya cumple `app-icon` hoy (spec.md, Edge Cases).

## Reglas de uso (no forman parte del componente, sino de cómo se invoca)

- Para un botón de solo ícono (sin texto visible), preferir etiquetar el elemento interactivo que lo
  envuelve (p. ej. `<button aria-label="Editar mesa">`) en vez de duplicar la etiqueta en el ícono,
  salvo que el ícono no esté envuelto por un elemento etiquetable (p. ej. un indicador de estado
  suelto dentro de una celda de tabla), caso en el cual se usa `ariaLabel` directamente sobre el
  ícono.
- Los indicadores de estado que combinan color y símbolo (p. ej. ocupada/libre) conservan la clase de
  color existente sobre el nuevo ícono, para no perder la semántica de color ya validada (FR-008).

## Fuera de este contrato

- Migrar los `name` existentes de `app-icon` uno por uno a este nuevo componente es trabajo de
  `tasks.md`, guiado por el catálogo de `data-model.md`.
- El mecanismo interno de autoalojamiento de la fuente de Material Icons (archivos, `@font-face`,
  clase CSS) está definido en `research.md` (Decisión D1) y no es parte del contrato público del
  componente — es un detalle de implementación que puede cambiar sin romper este contrato.
