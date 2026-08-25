# Implementation Plan: Rediseño de Layout de la Terminal de Mesas — Franja de Órdenes y Menú Central

**Branch**: `036-terminal-mesas-rediseno-layout` | **Date**: 2026-08-25 | **Spec**: [spec.md](./spec.md)

**Input**: Feature specification from `/specs/036-terminal-mesas-rediseno-layout/spec.md`

## Summary

Reorganizar el layout de la pantalla "Terminal de Mesas" (`pos-heladeria`, módulo `tables`) para que
coincida con el diseño de referencia, sin cambiar ninguna regla de negocio ya implementada (spec 028):
(1) el panel izquierdo de mesas por estado se convierte en una franja horizontal superior de "Órdenes
Recientes" con 3 filtros (Todas / Domicilios [vacío] / Mesas) y navegación tipo carrusel; (2) el
catálogo de productos deja de ser un panel superpuesto (`pos-catalog-drawer.component.ts`) y se
incrusta de forma permanente en el panel central, con buscador por nombre añadido a la ya existente
filtración por categoría; (3) se agrega un botón para colapsar/expandir el menú de navegación global
del dashboard, lo que requiere extender `sidebar.component.ts`/`layout.service.ts` con un
comportamiento de colapso de escritorio que hoy no existe (hoy el signal `sidebarOpen` solo controla
el slide-over móvil). El panel derecho de cobro (`pos-checkout-panel.component.ts`) no cambia de
comportamiento, solo de disposición visual. El filtro "Domicilios" se agrega como una pestaña vacía:
construir la creación real de órdenes de ese tipo requeriría un campo de backend y un flujo de "venta
de mostrador" que no existen hoy en el código, y se especifica en una spec futura independiente (ver
spec.md, Clarifications, sesión durante `/speckit-plan`).

## Technical Context

**Language/Version**: TypeScript 5.9.2, Angular 21.1 (componentes standalone, control flow `@if`/`@for`, signals)

**Primary Dependencies**: `@angular/core`/`@angular/cdk` 21.x, `@tanstack/angular-query-experimental` v5 (data fetching), RxJS 7.8, Tailwind CSS v4 (utilidades inline en `template:`, sin `.scss` por componente)

**Storage**: N/A — esta spec no agrega ni modifica entidades de backend (spec.md, Key Entities); reutiliza las mismas fuentes de datos ya consumidas por `pos-terminal.store.ts` (API de mesas/órdenes y de catálogo)

**Testing**: Vitest vía el builder `@angular/build:unit-test`; specs colocados `*.component.spec.ts` / `*.service.spec.ts`

**Target Platform**: Web — SPA Angular servida al navegador de la terminal POS (escritorio, pantalla ancha)

**Project Type**: Aplicación web existente (frontend Angular `pos-heladeria`); sin cambios de backend en esta spec

**Performance Goals**: Sin objetivos numéricos nuevos; el scroll tipo carrusel y el filtrado de búsqueda/categoría deben sentirse instantáneos (sin parpadeo ni recarga de red) igual que la interacción ya existente del catálogo y las mesas

**Constraints**: Cero cambios de comportamiento en validación de pago QR, cobro manual, cálculo de cambio, envío a cocina, facturación o cierre de mesa (spec 028, reforzado por FR-011/SC-004 de esta spec); no se toca ningún test con prefijo `"CONGELA comportamiento actual:"` (no existen en el módulo `tables` del frontend — confirmado, ver research.md); no se agregan dependencias nuevas (Principio IX) — el carrusel y el buscador se implementan con Angular/RxJS/Tailwind ya presentes

**Scale/Scope**: Una sola pantalla (`table-sessions.component.ts` y sus componentes hijos bajo `src/app/modules/tables/`); toca directamente 3 componentes existentes (`pos-tables-panel.component.ts`, `pos-catalog-drawer.component.ts`, `sidebar.component.ts`) más `pos-terminal.store.ts` y, en menor medida, `dashboard-layout.component.ts`/`layout.service.ts`

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

- **Principio I (Nace de un spec)** ✅ — `specs/036-terminal-mesas-rediseno-layout/spec.md` existe y fue clarificado antes de este plan.
- **Principio II (Comportamiento existente protegido)** ✅ — el spec documenta explícitamente que esto es una reorganización visual; ninguna regla de negocio de spec 028 cambia (FR-004/FR-008/FR-011, SC-004). No se requiere entrada en `registro-de-anomalias.md` porque no hay decisión de negocio que cambie comportamiento, solo presentación. La única pieza nueva de comportamiento (colapsar el sidebar global) es aditiva, no modifica ninguna regla existente.
- **Principio III (Characterization tests)** ✅ — no existen tests `"CONGELA comportamiento actual:"` en `pos-heladeria/src/app/modules/tables/` (confirmado por búsqueda); los únicos existen en el backend (`pos-backend/app/characterization_tests/`) y esta spec no toca backend. Gate trivialmente satisfecho.
- **Principio IV (Nuevo comportamiento vía spec)** ✅ — el único comportamiento nuevo (toggle de sidebar global, búsqueda por nombre en catálogo) está definido en el spec (FR-006, FR-010).
- **Principio V (No refactors oportunistas)** ✅ — el plan solo toca los archivos necesarios para las 3 historias de usuario; no se incluye ninguna refactorización no relacionada.
- **Principio VI (Evolución incremental)** ✅ — esta es la aplicación directa del principio: el hallazgo de que "Domicilio" mezclaría nueva funcionalidad + migración de datos + layout llevó a acotar esta spec a un solo tipo de cambio (layout/presentación) y diferir el resto a una spec futura (ver spec.md, Clarifications).
- **Principio VII (Datos históricos)** N/A — no se toca facturación ni datos históricos.
- **Principio VIII (Evolución del modelo de datos)** N/A — no hay cambios de modelo de datos en esta spec (ver Key Entities del spec: "no se le agregan atributos nuevos").
- **Principio IX (Dependencias nuevas)** ✅ — no se agrega ninguna dependencia; el carrusel se implementa con `scrollBy` nativo + Angular signals (ver research.md), no con una librería de carrusel.
- **Principio X (Verificación obligatoria)** — pendiente de ejecución en `/speckit-tasks` + `/speckit-implement`: se exige mantener en verde `pos-checkout-panel.component.spec.ts` y `pos-terminal.store.spec.ts` ya existentes, y agregar specs nuevos para los componentes sin cobertura hoy (`pos-tables-panel.component.ts`, el nuevo panel de catálogo embebido).
- **Principio XI (Negocio vs. técnico)** ✅ — la decisión de negocio de acotar "Domicilio" a un filtro vacío quedó registrada explícitamente en el spec (Clarifications), no decidida unilateralmente en el plan.
- **Principio XII (Trazabilidad)** ✅ — Necesidad (imagen de referencia del usuario) → Spec 036 → Decisiones (Clarifications) → este Plan → Tasks/Implementación/Tests (siguientes comandos).
- **Principio XIII (Español de Colombia)** ✅ — todo este documento y los artefactos generados se redactan en español de Colombia.

**Resultado**: Gate PASA sin violaciones. No se requiere la tabla de Complexity Tracking.

**Re-chequeo post-diseño (tras Fase 1)**: research.md, data-model.md y contracts/ui-store-contract.md
confirman que el diseño no agrega dependencias nuevas (carrusel implementado nativamente — Principio
IX), no modifica el modelo de datos de backend (Principio VIII, N/A), no toca ningún characterization
test (Principio III), y mantiene el comportamiento de cobro/validación/facturación intacto (Principio
II). Único ajuste de alcance detectado durante el diseño: `sidebar.component.ts` necesita honrar
`sidebarOpen()` también en escritorio (hoy solo aplica en móvil) — es una extensión acotada de un
componente ya existente, no una nueva pieza de arquitectura. Gate sigue PASANDO sin violaciones.

## Project Structure

### Documentation (this feature)

```text
specs/036-terminal-mesas-rediseno-layout/
├── plan.md              # Este archivo (/speckit-plan)
├── research.md          # Fase 0 (/speckit-plan)
├── data-model.md        # Fase 1 (/speckit-plan)
├── quickstart.md        # Fase 1 (/speckit-plan)
├── contracts/           # Fase 1 (/speckit-plan) — contratos de UI/store, sin API nueva
│   └── ui-store-contract.md
└── tasks.md             # Fase 2 (/speckit-tasks — NO se crea en este comando)
```

### Source Code (repositorio de la aplicación)

El código vive en el repositorio hermano `../pos-heladeria` (Angular), no en este repositorio de specs
(`pos-specs`). No se requiere ningún cambio en `../pos-backend` para esta spec (ver Constraints).

```text
pos-heladeria/src/app/modules/tables/
├── pages/
│   └── table-sessions.component.ts      # Layout raíz de la pantalla (3 columnas → franja superior + centro + derecha)
├── components/
│   ├── pos-tables-panel.component.ts    # HOY: grilla de mesas por estado → SE CONVIERTE en franja "Órdenes Recientes" con filtros y carrusel
│   ├── pos-catalog-drawer.component.ts  # HOY: overlay de pantalla completa → SE EMBEBE en el panel central (se retira el wrapper `fixed inset-0 bg-black/40`)
│   ├── pos-checkout-panel.component.ts  # Sin cambio de comportamiento; solo ajuste de disposición visual (FR-008)
│   ├── pos-order-panel.component.ts     # Sin cambio (constructor de orden manual, ya en el centro)
│   ├── payment-validation-block.component.ts  # Sin cambio de comportamiento
│   └── manual-order-panel.component.ts  # Sin cambio de comportamiento
└── services/
    └── pos-terminal.store.ts            # Se agregan: signal de pestaña de filtro (todas/domicilios/mesas), signal de texto de búsqueda de catálogo, computed de catálogo filtrado por nombre+categoría, computed de la franja de "Órdenes Recientes"

pos-heladeria/src/app/modules/dashboard/layout/
├── layout.service.ts        # Ya expone sidebarOpen()/toggle(); SIN cambios de API, se reutiliza tal cual
├── sidebar.component.ts     # SE EXTIENDE: hoy fuerza `md:relative md:translate-x-0` (visible siempre en escritorio); se hace ese comportamiento condicional a sidebarOpen() también en escritorio
└── dashboard-layout.component.ts  # Se ajusta el margen/ancho del contenido cuando el sidebar está colapsado en escritorio
```

**Structure Decision**: Se modifica in-place el módulo `tables` ya existente en `pos-heladeria` (no se
crea un módulo nuevo ni una segunda app). El único archivo compartido fuera de `tables/` que se toca es
el par `layout.service.ts`/`sidebar.component.ts` del módulo `dashboard`, porque el botón de
colapsar/expandir vive en la Terminal de Mesas pero controla el menú de navegación global (decisión
registrada en spec.md, Clarifications).

## Complexity Tracking

*Sin violaciones que justificar — el Constitution Check pasa limpio (ver arriba).*
