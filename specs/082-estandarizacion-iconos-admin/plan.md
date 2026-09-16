# Implementation Plan: Estandarización de Íconos en el Panel de Administración

**Branch**: `082-estandarizacion-iconos-admin` | **Date**: 2026-09-16 | **Spec**: [spec.md](./spec.md)

**Input**: Feature specification from `/specs/082-estandarizacion-iconos-admin/spec.md`

## Summary

En `pos-heladeria` (Angular), el panel de administración muestra hoy sus íconos mediante dos
sistemas inconsistentes: un componente SVG artesanal (`app-icon`, `src/app/shared/icon/icon.component.ts`,
35 nombres, rutas Lucide copiadas a mano) y emoji sueltos usados como ícono en ~30-40 archivos de los
módulos de panel principal, caja, inventario, mesas/terminal de personal, usuarios, configuración y
promociones — incluida la temática de heladería (🍦) en la marca por defecto del panel lateral y en
la tarjeta "Productos" del dashboard. Este plan introduce un componente Angular nuevo (reutilizable,
con soporte de `aria-label` para accesibilidad) respaldado por la librería Material Icons —
autoalojada, variante Outlined, para que siga funcionando sin conexión— y migra a él todos los
íconos que hoy se muestran dentro del panel de administración, dejando fuera del cambio el flujo
público de menú QR por código (FR-010). La investigación de código real reveló que `app-icon` y
algunos componentes con emoji también sirven a ese flujo público; la resolución de ese hallazgo
(no se borra `app-icon`; los componentes compartidos sí se migran como efecto visual secundario) ya
quedó registrada en `spec.md` (Clarifications, sesión durante `/speckit-plan`) y se detalla en
`research.md` (Decisiones D2 y D6). Sin cambios de backend, sin migraciones de datos, sin
dependencias nuevas en `pos-backend` — una única dependencia nueva en `pos-heladeria` (`material-icons`,
justificada en el Constitution Check, Principio IX).

## Technical Context

**Language/Version**: TypeScript ~5.9.2, Angular ^21.1.0 (componentes standalone, `signal`/`computed`,
sin `NgModule`) — solo `pos-heladeria`; `pos-backend` no se toca.

**Primary Dependencies**: `@angular/core`/`@angular/cdk` ^21.1.x, `@tanstack/angular-query-experimental`
v5, RxJS 7.8, Tailwind CSS v4 (`@tailwindcss/postcss`, CSS-first, sin `tailwind.config.*`),
`@angular/service-worker` ^21.1.0 (ya existente, sin cambios de configuración — ver Storage abajo).
Dependencia **nueva**: `material-icons` (npm, fuente autoalojable, variante Outlined) — justificación
completa en Constitution Check (Principio IX) y en `research.md`, Decisión D1.

**Storage**: N/A — sin persistencia nueva; el "catálogo de íconos" (`data-model.md`) es una tabla de
correspondencia estática nombre→ícono, no una entidad de datos. El service worker ya existente
(`ngsw-config.json`, grupo `lazy` con glob `woff|woff2|otf|ttf`) cachea automáticamente los archivos
de fuente nuevos colocados en `public/fonts/material-icons/` — no requiere cambios de configuración.

**Testing**: `ng test` (Vitest vía el builder `@angular/build:unit-test`), specs colocados
(`*.component.spec.ts`). No existe script `lint` ni configuración de ESLint en este repositorio — no
hay gate de lint que integrar. Búsqueda de `"CONGELA comportamiento actual:"` en todo `src/app`:
0 coincidencias — no hay characterization tests protegidos en riesgo (Principio III).

**Target Platform**: Aplicación web (SPA Angular) servida al navegador, con service worker para modo
offline ya configurado (spec 077/028) — el requisito de FR-001/SC-007 (íconos disponibles sin
conexión) se apoya en ese mecanismo ya existente, sin construir uno nuevo.

**Project Type**: Aplicación web de 2 repositorios ya existentes y en producción (`pos-backend` +
`pos-heladeria`), consistente con el Alcance de la Constitución. Esta feature toca **solo**
`pos-heladeria` (frontend); no hay cambios en `pos-backend`.

**Performance Goals**: Sin objetivo de performance nuevo. La fuente autoalojada de la variante
Outlined es un único archivo de peso moderado, cacheado por el service worker existente tras la
primera carga — sin round-trip de red en cargas subsiguientes ni en modo offline.

**Constraints**:
- No modificar el comportamiento, rutas ni reglas de negocio del flujo público de menú QR (FR-010).
- No eliminar ni modificar `src/app/shared/icon/icon.component.ts` (`app-icon`) — sigue sirviendo,
  intacto, a ese flujo público (research.md, D2).
- Preservar el patrón de theming por herencia de color/tamaño del elemento padre que ya usan los
  íconos actuales (`currentColor`/clases Tailwind) — ninguna pantalla migrada debe cambiar su forma
  de controlar tamaño o color de ícono.
- No introducir `@angular/material` completo ni ningún sistema de theming/componentes ajeno —
  únicamente la fuente de íconos (Principio IX, simplicidad).

**Scale/Scope**: 7 pantallas/módulos del panel de administración (dashboard, caja, inventario,
mesas/terminal de personal, usuarios, configuración, promociones); ~35 nombres de `app-icon` +
~30 usos de emoji a migrar dentro de esas pantallas (catálogo completo en `data-model.md`); 4
componentes compartidos con el flujo público que también se migran como efecto visual secundario
(research.md, D6). 1 componente Angular nuevo, 1 dependencia npm nueva.

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

- **Principio I (Nace de un spec)** — PASS. `spec.md` existe, fue aprobado y pasó por dos sesiones de
  Clarifications (inicial y durante este mismo plan) antes de continuar.
- **Principio II (Comportamiento existente protegido)** — PASS con justificación inline. Esta spec sí
  cambia comportamiento visible (reemplazo de íconos, incluido un efecto visual secundario en
  componentes compartidos con el flujo público, research.md D6), pero es un cambio de presentación,
  no de reglas de negocio, y la decisión completa (quién: usuario que solicitó la funcionalidad;
  cuándo: 2026-09-16; qué cambia; por qué; qué se ve afectado) ya queda trazada dentro de la propia
  `spec.md` (User Stories + Clarifications) — mismo criterio que specs 059/076 aplicaron para no
  requerir una entrada adicional en `registro-de-anomalias.md` cuando la autorización ya queda
  documentada dentro de la spec misma.
- **Principio III (Characterization tests)** — PASS. Se verificó (grep) que ningún archivo
  `*.spec.ts` de todo `pos-heladeria` usa el prefijo `"CONGELA comportamiento actual:"` — no existe
  ningún test protegido que esta funcionalidad ponga en riesgo.
- **Principio IV (Nuevo comportamiento permitido)** — PASS. Aplica directamente: el criterio de éxito
  es conformidad con `spec.md`, no equivalencia visual con el sistema anterior.
- **Principio V (No mezclar refactors oportunistas)** — PASS, con atención en Fase 2: al tocar
  `sidebar.component.ts`, `admin-dashboard.component.ts` y los demás archivos listados en
  `data-model.md`, las tareas deben limitarse al reemplazo de íconos — ninguna lógica no relacionada
  de esos componentes se reescribe como parte de esta feature.
- **Principio VI (Evolución incremental)** — PASS. Las 3 historias de usuario son independientes y
  priorizadas (P1, P1, P2); además, dentro de la Historia 1, cada módulo (dashboard, caja, inventario,
  mesas, usuarios, configuración, promociones) es una unidad de migración verificable por separado.
- **Principio VII (Datos históricos)** — PASS, no aplica: no se toca ninguna venta ni factura emitida.
- **Principio VIII (Evolución del modelo de datos)** — PASS, no aplica: cero migraciones, cero
  entidades ni columnas nuevas (ver Storage arriba).
- **Principio IX (Dependencias nuevas)** — PASS, con justificación. Se introduce `material-icons`
  (npm) en `pos-heladeria`. *Problema que resuelve*: proveer una librería de íconos estándar,
  autoalojable, ya que el proyecto no tiene ninguna instalada hoy y la funcionalidad la pide
  explícitamente. *Por qué no basta la librería estándar*: ni Angular ni Node incluyen un conjunto de
  íconos. *Alternativas consideradas*: Google Fonts CDN (rechazada — requiere conexión, viola
  FR-001/SC-007), `@angular/material` completo (rechazada — trae un sistema de theming/componentes
  no solicitado, viola simplicidad), Material Symbols (rechazada — no es lo pedido explícitamente,
  añade complejidad de fuente variable innecesaria). Detalle completo en `research.md`, Decisión D1.
  *Impacto en mantenimiento/seguridad/despliegue*: paquete de solo activos estáticos (fuente + CSS),
  sin código ejecutable de terceros en runtime más allá de esos archivos; se integra al pipeline de
  build/assets ya existente sin nueva configuración de CI ni del service worker.
- **Principio X (Verificación obligatoria)** — Pendiente de Fase 2 (tasks): cada módulo migrado
  necesita un test que confirme ausencia de `app-icon`/emoji y presencia del nuevo componente, más
  los escenarios de `quickstart.md`. No bloquea el gate de este plan.
- **Principio XI (Negocio vs. técnico)** — PASS. Las decisiones de negocio (qué se reemplaza, qué
  queda fuera de alcance) ya están tomadas en `spec.md`; este plan solo traduce eso a una estrategia
  técnica (librería, componente, catálogo).
- **Principio XII (Trazabilidad)** — PASS. Cadena completa: Necesidad (pedido del usuario) → Spec 082
  (+ 2 sesiones de Clarifications) → Decisión (este plan + `research.md`) → Tasks (siguiente comando)
  → Tests.
- **Principio XIII (Español de Colombia)** — PASS. Este plan y los artefactos de Fase 0/1 se redactan
  en español de Colombia, igual que `spec.md`.

**Resultado del gate**: sin violaciones — no se requiere ninguna entrada en `Complexity Tracking`
(la única dependencia nueva ya quedó justificada arriba, como exige el Principio IX, no como
excepción).

## Project Structure

### Documentation (this feature)

```text
specs/082-estandarizacion-iconos-admin/
├── plan.md                                    # This file (/speckit-plan command output)
├── research.md                                # Phase 0 output (/speckit-plan command)
├── data-model.md                              # Phase 1 output (/speckit-plan command) — catálogo de íconos
├── quickstart.md                              # Phase 1 output (/speckit-plan command)
├── contracts/
│   └── icon-component-contract.md             # Phase 1 output (/speckit-plan command)
└── tasks.md                                   # Phase 2 output (/speckit-tasks command - NOT created by /speckit-plan)
```

### Source Code (repositorio existente, fuera de `pos-specs`)

Este proyecto de specs (`pos-specs`) no contiene código fuente: la implementación vive en
`pos-heladeria` (Angular). `pos-backend` no se toca. La estructura relevante ya existe — esta
feature no crea módulos nuevos, solo un componente y los archivos de fuente autoalojados:

```text
../pos-heladeria/                                    # Angular 21 (standalone, signals) + Tailwind 4
├── public/
│   └── fonts/material-icons/                        # NUEVO — archivos de fuente Outlined autoalojados
├── src/
│   ├── styles.css                                    # se agrega @font-face + clase .material-icons-outlined
│   ├── app/
│   │   ├── shared/
│   │   │   ├── icon/
│   │   │   │   └── icon.component.ts                 # `app-icon` — SIN CAMBIOS (research.md, D2)
│   │   │   └── icon-mi/                               # NUEVO — componente de ícono reutilizable (contracts/icon-component-contract.md)
│   │   │       └── icon-mi.component.ts
│   │   └── modules/
│   │       ├── dashboard/
│   │       │   ├── layout/sidebar.component.ts        # 🍦/🛡️ → nuevo componente (data-model.md)
│   │       │   └── pages/admin-dashboard.component.ts # tiles/quick actions con emoji → nuevo componente
│   │       ├── cash-register/                         # ✕ en modales → nuevo componente
│   │       ├── inventory/                              # ✕ en modal → nuevo componente
│   │       ├── users/                                  # 👥/✕ → nuevo componente
│   │       ├── settings/                               # tenant-info.component.ts (🎨🏪🧾🖨️✓👤) → nuevo componente
│   │       ├── tables/
│   │       │   ├── pages/tables-page.component.ts      # 🪑📷✏️🔴🟢 → nuevo componente (mesas/terminal, admin)
│   │       │   ├── pages/table-sessions.component.ts   # ✅ → nuevo componente
│   │       │   ├── pages/manual-order-page.component.ts # usa `app-icon` y los componentes compartidos de abajo
│   │       │   └── components/
│   │       │       ├── pos-order-panel.component.ts     # 🍽️📍📞🛵✓ → nuevo componente
│   │       │       ├── pos-checkout-panel.component.ts  # ✏️🧾🔓 → nuevo componente
│   │       │       ├── pos-tables-panel.component.ts    # 🧾 → nuevo componente
│   │       │       ├── cart.component.ts                # COMPARTIDO con menú público — 🛒✕ → nuevo componente (research.md D6)
│   │       │       ├── product-select.component.ts      # COMPARTIDO — 🏷️ → nuevo componente (D6)
│   │       │       ├── payment-attempt-review-panel.component.ts # COMPARTIDO — 💳 → nuevo componente (D6)
│   │       │       └── pos-catalog-drawer.component.ts  # COMPARTIDO — 🏷️ → nuevo componente (D6)
│   │       └── tables/pages/
│   │           ├── public-menu.component.ts             # FUERA DE ALCANCE — sin cambios (usa `app-icon`)
│   │           ├── diner-shell.component.ts              # FUERA DE ALCANCE — sin cambios
│   │           ├── expired-qr.component.ts                # FUERA DE ALCANCE — sin cambios
│   │           └── checkout/*-step.component.ts            # FUERA DE ALCANCE — sin cambios (usa `app-icon`)
│   └── app.routes.ts                                    # sin cambios — solo referencia para research.md D5
└── package.json                                          # se agrega `material-icons` como dependencia nueva
```

**Structure Decision**: se reutiliza íntegramente la estructura de módulos ya existente en
`pos-heladeria`; no se crea ningún módulo nuevo. Se agrega un único componente nuevo
(`shared/icon-mi/` o ubicación equivalente definida en Fase 2) y los archivos de fuente autoalojados
bajo `public/fonts/`. El componente artesanal existente (`shared/icon/icon.component.ts`) permanece
intacto para seguir sirviendo al flujo público (research.md, D2) — no se mueve, no se renombra, no
se borra. Las tareas de Fase 2 (`tasks.md`) se organizan por módulo (uno por historia/unidad
verificable, Principio VI), usando el catálogo de `data-model.md` como fuente de verdad de qué
nombre semántico mapea a qué ícono de Material Icons en cada archivo.

## Constitution Check (post-diseño)

Re-evaluado tras Fase 1 (`research.md`, `data-model.md`, `contracts/icon-component-contract.md`,
`quickstart.md`): el diseño no introduce ninguna entidad, columna, migración ni dependencia adicional
a la ya justificada en el gate inicial (`material-icons`, Principio IX). El único ajuste respecto al
gate inicial es la confirmación, vía investigación de código real, de que el componente artesanal y
cuatro componentes de la Terminal de Mesas son compartidos con el flujo público — ya incorporado a
`spec.md` (Clarifications) y a este plan (Constitution Check, Principio II, y Project Structure
arriba). El gate se mantiene — **sin violaciones**.

## Complexity Tracking

*Sin violaciones del Constitution Check — tabla no aplica. La única dependencia nueva
(`material-icons`) está justificada arriba bajo el Principio IX, no como una excepción sino como el
procedimiento normal que ese principio exige para dependencias nuevas.*
