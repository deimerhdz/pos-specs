# Implementation Plan: Imagen de producto en el catálogo y organización del tipo de orden — creación de orden manual

**Branch**: `051-imagen-producto-tipo-orden` | **Date**: 2026-08-28 | **Spec**: [spec.md](./spec.md)

**Input**: Feature specification from `/specs/051-imagen-producto-tipo-orden/spec.md`

## Summary

Sobre la pantalla de creación de orden manual (`pos-heladeria`, módulo `tables`,
`manual-order-page.component.ts`): la tarjeta de cada producto en la grilla del catálogo
(líneas 125-137) pasa a mostrar la imagen del producto (`p.image_url`, ya expuesto por
`MenuProduct` y ya usado por `product-select.component.ts` en el detalle) con el mismo criterio
visual — contenedor `aspect-square` + ícono `image-off` de respaldo — que ya usa el catálogo del
menú público por QR (`public-menu.component.ts:359-372`); el detalle/opciones
(`product-select.component.ts:48-51`) ya muestra la imagen hoy y no requiere cambios. Además, se
agrega un título propio inmediatamente encima del listado horizontal de mesas (línea 74), separado
en jerarquía visual del `<h2>Tipo de Orden</h2>` ya existente (línea 44), para que ambos
encabezados se lean como secciones distintas. Un único archivo de producción modificado, sin
cambios de store, sin cambios de backend, sin dependencias nuevas — ver `research.md` (decisiones
D1-D4).

## Technical Context

**Language/Version**: TypeScript 5.9.2, Angular 21.1 (componentes standalone, control flow
`@if`/`@for`, signals)

**Primary Dependencies**: `@angular/core` 21.1.x — sin dependencias nuevas; se agrega
`IconComponent` (`shared/icon/icon.component.ts`, ya existente, ya usado en `public-menu.component.ts`)
a los `imports` de `ManualOrderPageComponent`

**Storage**: N/A — no se agrega ni modifica ninguna entidad de backend ni de frontend
(data-model.md); no se toca ningún endpoint de `pos-backend`

**Testing**: Vitest vía el builder `@angular/build:unit-test`; specs colocados
`*.component.spec.ts`

**Target Platform**: Web — SPA Angular servida al navegador de la terminal POS (escritorio,
pantalla ancha)

**Project Type**: Aplicación web existente (frontend Angular `pos-heladeria`); sin cambios de
backend en esta spec

**Performance Goals**: Sin objetivos numéricos nuevos; el único trabajo adicional es cargar las
imágenes ya referenciadas por `image_url` (mismas URLs que ya carga el detalle de producto), sin
llamadas de red ni endpoints nuevos

**Constraints**: Cero cambios en la lógica de selección/configuración de producto
(`store.openConfig`, `store.addDraftFromSelection`), en la disponibilidad de "Para Llevar"/
"Domicilio" (siguen deshabilitadas), en qué mesas se listan o su estado, y en el cálculo de
subtotal/impuesto/total (spec.md, Assumptions/FR-006); 0 tests con prefijo `"CONGELA comportamiento
actual:"` en `pos-heladeria/src/` (research.md, confirmado igual que specs 045/046/047/048/049); no
se agregan dependencias nuevas (Principio IX)

**Scale/Scope**: Un único archivo de producción,
`pos-heladeria/src/app/modules/tables/pages/manual-order-page.component.ts` — se modifica la
tarjeta de catálogo (líneas 125-137: contenedor de imagen + reubicación del badge de descuento y
del bloque de texto) y se agrega un encabezado nuevo antes del listado de mesas (línea 74); se
agrega `IconComponent` a `imports`. `product-select.component.ts` no se modifica (research.md D3).
Tests nuevos/ajustados en `manual-order-page.component.spec.ts` (ninguno de los 8 casos existentes
depende del markup tocado, research.md). Ningún otro componente, servicio ni endpoint de backend se
modifica.

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

- **Principio I (Nace de un spec)** ✅ — `specs/051-imagen-producto-tipo-orden/spec.md` existe, sin
  `[NEEDS CLARIFICATION]` pendiente, antes de este plan.
- **Principio II (Comportamiento existente protegido)** ✅ — spec.md documenta el estado actual
  exacto (tarjeta sin imagen, listado de mesas sin título propio) con autorización directa del
  dueño/desarrollador el 2026-08-28. No reabre ninguna regla de precio, inventario, disponibilidad
  de tipos de orden ni facturación; no aplica una nueva entrada en `registro-de-anomalias.md`
  (mismo criterio que specs 045/048/049, ajuste de visualización sobre una pantalla existente).
- **Principio III (Characterization tests)** ✅ — 0 tests `"CONGELA comportamiento actual:"` en
  `pos-heladeria/src/` (research.md, mismo hallazgo que specs 045-049).
- **Principio IV (Nuevo comportamiento vía spec)** ✅ — todo el comportamiento nuevo (imagen en
  tarjeta de catálogo, título sobre el listado de mesas) está definido en spec.md, FR-001 a
  FR-006.
- **Principio V (No refactors oportunistas)** ✅ — solo se toca el archivo y las líneas
  directamente causadas por FR-001/FR-002/FR-004/FR-005; `product-select.component.ts` se deja
  intacto porque ya cumple FR-003 (research.md D3); no se reescribe ningún otro bloque del
  componente (buscador, categorías, carrito, resumen de totales).
- **Principio VI (Evolución incremental)** ✅ — un solo tipo de cambio (visualización de UI sobre
  un archivo ya existente), sin migración de datos, sin cambio de arquitectura ni de backend.
- **Principio VII (Datos históricos)** N/A — no se toca facturación ni se recalcula ninguna venta
  ya cerrada.
- **Principio VIII (Evolución del modelo de datos)** N/A — data-model.md: sin entidades ni campos
  nuevos; se reutiliza `MenuProduct.image_url`, ya existente y ya poblado.
- **Principio IX (Dependencias nuevas)** ✅ — no se agrega ninguna dependencia; `IconComponent` ya
  existe en el repositorio y ya se usa para el mismo propósito en `public-menu.component.ts`.
- **Principio X (Verificación obligatoria)** — pendiente de ejecución en `/speckit-tasks` +
  `/speckit-implement`: mantener en verde los 8 casos existentes de
  `manual-order-page.component.spec.ts`; agregar cobertura nueva para la imagen (con y sin
  `image_url`) en la tarjeta del catálogo y para la presencia del título sobre el listado de mesas
  (quickstart.md, Escenarios 1 y 3).
- **Principio XI (Negocio vs. técnico)** ✅ — la necesidad de negocio (mostrar imagen del producto,
  organizar el tipo de orden) viene directamente del dueño/desarrollador en spec.md, Input; las
  decisiones de este documento (D1-D4) son todas técnicas (qué patrón visual y qué archivo tocar,
  no qué debe mostrarse).
- **Principio XII (Trazabilidad)** ✅ — Necesidad (pedido directo del dueño/desarrollador) → Spec
  051 → este Plan → Tasks/Implementación/Tests (siguientes comandos).
- **Principio XIII (Español de Colombia)** ✅ — todo este documento y los artefactos generados se
  redactan en español de Colombia.

**Resultado**: Gate PASA sin violaciones. No se requiere la tabla de Complexity Tracking.

**Re-chequeo post-diseño (tras Fase 1)**: `data-model.md` confirma que el diseño no agrega
dependencias nuevas (Principio IX), no modifica el modelo de datos de backend (Principio VIII,
N/A), no toca ningún characterization test (Principio III, 0 en `pos-heladeria/src/`), y no altera
ningún dato histórico ni regla de cálculo de facturación (Principio VII) — únicamente muestra un
dato (`image_url`) que el backend ya entrega y agrega texto estático de UI. Gate sigue PASANDO sin
violaciones.

## Project Structure

### Documentation (this feature)

```text
specs/051-imagen-producto-tipo-orden/
├── plan.md              # Este archivo (/speckit-plan)
├── research.md          # Fase 0 (/speckit-plan) — decisiones D1-D4
├── data-model.md        # Fase 1 (/speckit-plan) — sin entidades nuevas
├── quickstart.md        # Fase 1 (/speckit-plan) — validación manual, 3 escenarios
└── tasks.md             # Fase 2 (/speckit-tasks) — aún no generado
```

No se genera `contracts/`: esta spec no expone ni consume ninguna API HTTP nueva ni modificada, no
agrega ni modifica ningún método/computed del store, y no hay ningún contrato de UI/store nuevo que
documentar más allá de leer un campo (`image_url`) que el componente ya recibe y agregar un
encabezado estático — mismo criterio que spec 047.

### Source Code (repositorio de la aplicación)

El código vive en el repositorio hermano `../pos-heladeria` (Angular), no en este repositorio de
specs (`pos-specs`). No se requiere ningún cambio en `../pos-backend` para esta spec.

```text
pos-heladeria/src/app/modules/tables/
├── pages/
│   ├── manual-order-page.component.ts       # HOY: tarjeta de catálogo (~125-137) con solo
│   │                                          # nombre/precio → SE AGREGA contenedor de imagen
│   │                                          # (`p.image_url` con `<img>`/ícono `image-off` de
│   │                                          # respaldo, patrón de `public-menu.component.ts`) y
│   │                                          # se reubica el badge de descuento y el bloque de
│   │                                          # texto (research.md D1-D2); bloque de tipo de orden
│   │                                          # + mesas (~34-94) SIN título propio sobre el
│   │                                          # listado de mesas (~74) → SE AGREGA un `<h3>` con
│   │                                          # menor jerarquía visual que el `<h2>Tipo de
│   │                                          # Orden</h2>` existente (~44) (research.md D4); se
│   │                                          # agrega `IconComponent` a `imports`
│   └── manual-order-page.component.spec.ts  # Se agregan casos nuevos para imagen (con/sin
│                                              # `image_url`) y para el título del listado de
│                                              # mesas; los 8 casos existentes no se tocan
│                                              # (research.md, "Resumen de impacto en tests
│                                              # existentes")
└── components/
    └── product-select.component.ts          # Sin cambios — ya muestra `product.image_url`
                                                # (~48-51) para el detalle/opciones (research.md D3)
```

**Structure Decision**: Se modifica in-place un único archivo de producción ya existente
(`manual-order-page.component.ts`), sin crear ningún componente nuevo — se reutiliza el patrón
visual de imagen/placeholder ya existente en `public-menu.component.ts` y el `IconComponent`
compartido, en vez de introducir un componente de tarjeta de producto separado.

## Complexity Tracking

*Sin violaciones que justificar — el Constitution Check pasa limpio (ver arriba).*
