# Implementation Plan: Rediseño responsive de la Terminal de Mesas (escritorio/tablet/móvil)

**Branch**: `076-terminal-mesas-rediseno-responsive` | **Date**: 2026-09-04 | **Spec**: [spec.md](./spec.md)

**Input**: Feature specification from `/specs/076-terminal-mesas-rediseno-responsive/spec.md`

## Summary

Reemplazar el carrusel horizontal de una sola fila de la grilla de mesas
(`pos-tables-panel.component.ts:57-102`, spec 036) por una cuadrícula responsive que envuelve en
varias filas (4 columnas en escritorio ≥1024px, 3 en tablet 768–1023px, 2 en móvil <768px, mismos
breakpoints `md`/`lg` ya usados en `table-sessions.component.ts`), y agregar tres piezas de
funcionalidad nueva ya acotadas por el negocio (spec.md, Clarifications): contadores de ocupación
(sin datos nuevos, solo agregación de lo ya cargado), una barra superior operativa (turno de caja,
reloj, indicador de sincronización, acceso directo F1 a `openShift()` ya existente en el módulo de
Caja), y una referencia "Atendido por" en la tarjeta de mesa que expone un campo que el backend ya
captura hoy (`DiningOrder.user_id`) pero que ningún endpoint ni pantalla expone todavía. El resto
del comportamiento ya implementado (paneles central y de cobro con selección activa, atajos F2/F3/
ESC/Ctrl+P, diálogo de éxito) no cambia — este plan solo reordena y hace responsive el contenedor
visual alrededor de ese comportamiento.

## Technical Context

**Language/Version**: TypeScript ~5.9.2 (frontend, Angular standalone components + signals) /
Python 3.14 (backend, FastAPI + SQLAlchemy + Pydantic 2.13)

**Primary Dependencies**: Angular ^21.1.0 (componentes standalone, `signal`/`computed`, sin
`NgModule`) y Tailwind CSS ^4.1.12 (clases utilitarias, incluida la paleta `STATUS_META` ya
existente que esta spec preserva) en `pos-heladeria`; FastAPI 0.136 + Pydantic 2.13 + SQLAlchemy en
`pos-backend`, sin dependencias nuevas (Principio IX — no se justifica ninguna para este alcance).

**Storage**: PostgreSQL 16, schema-per-tenant (ya existente). Sin migraciones: el campo que se
expone (`DiningOrder.user_id`, `app/models/customer_order.py:115`) ya existe en la tabla; esta
feature solo agrega su lectura/serialización (join a `shared.users`) al `OrderResponse` ya devuelto
por los endpoints de órdenes.

**Testing**: Frontend — `ng test` (Vitest ^4.0.8 + utilidades de testing de Angular), archivos
`*.spec.ts` junto a cada componente/store (`table-sessions.component.spec.ts`,
`pos-tables-panel.component.spec.ts`, `pos-terminal.store.spec.ts`, etc.). Backend — sin pytest:
scripts autoejecutables bajo `app/scripts/test_*.py` (`python -m app.scripts.test_X`), siguiendo el
mismo patrón que `test_table_release.py`. Ningún archivo de test de las áreas tocadas por esta spec
lleva hoy el prefijo `"CONGELA comportamiento actual:"` (verificado por búsqueda en
`src/app/modules/tables/**/*.spec.ts`) — no hay conflicto con el Principio III de la constitución.

**Target Platform**: Aplicación web (SPA Angular) servida al navegador; responsive dentro de la
misma build para escritorio, tablet y móvil — no hay build ni target nativo separado.

**Project Type**: Aplicación web de 2 repositorios ya existentes y en producción (`pos-backend` +
`pos-heladeria`), consistente con el Alcance de la Constitución. Esta feature toca ambos: exposición
de un campo ya existente en el backend, y UI/rediseño en el frontend.

**Performance Goals**: Sin objetivo de performance nuevo — SC-001/SC-006 son de percepción/uso
(scroll vertical en vez de horizontal, resumen visible de un vistazo), no de throughput o latencia;
se reutiliza el mismo mecanismo reactivo (`signal`/`computed`) ya usado por `tablesView()`.

**Constraints**: preservar exactamente el vocabulario de estado y la paleta de colores ya existentes
(FR-029/FR-030, `STATUS_META` en `pos-terminal.store.ts:115-123`); reutilizar los breakpoints `md`
(768px)/`lg` (1024px) ya usados en esta misma pantalla en vez de definir unos nuevos; cero
migraciones de base de datos (Out of Scope de spec.md).

**Scale/Scope**: 1 pantalla (Terminal de Mesas), 5 historias de usuario, 31 requisitos funcionales.
Frontend: 1 página (`table-sessions.component.ts`), 2 componentes existentes a modificar
(`pos-tables-panel.component.ts`, `order-summary-card.component.ts`), 1 store existente a extender
(`pos-terminal.store.ts`), 1 componente nuevo pequeño (barra superior). Backend: 1 schema de
respuesta a extender (`OrderResponse`, `app/api/v1/orders/schemas.py:197-224`), sin entidades ni
tablas nuevas.

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

- **Principio I (Nace de un spec)** — PASS. `spec.md` ya existe y fue aprobado (incluida sesión de
  Clarifications) antes de este plan.
- **Principio II (Comportamiento existente protegido)** — PASS con justificación inline. Esta spec sí
  cambia comportamiento visible (carrusel → cuadrícula; barra superior nueva) pero la decisión de
  negocio que lo autoriza (quién: dueño/desarrollador; cuándo: 2026-09-04; qué cambia; por qué; qué
  se ve afectado) ya está documentada íntegramente dentro de `spec.md` ("Autorización de negocio" +
  "Clarifications") — mismo criterio que spec 059 aplicó para no requerir una entrada adicional en
  `registro-de-anomalias.md` cuando la autorización ya queda trazada dentro de la propia spec.
- **Principio III (Characterization tests)** — PASS. Se verificó (grep) que ningún archivo
  `*.spec.ts` de los módulos tocados (`table-sessions`, `pos-tables-panel`, `pos-checkout-panel`,
  `pos-terminal.store`) usa el prefijo `"CONGELA comportamiento actual:"` sobre el carrusel, los
  colores de estado, o cualquier otro comportamiento que esta spec modifica — no hay ningún test
  protegido que deba tocarse bajo autorización especial.
- **Principio IV (Nuevo comportamiento permitido)** — PASS. Aplica directamente: el objetivo es
  conformidad con `spec.md`, no equivalencia total con el sistema anterior.
- **Principio V (No mezclar refactors oportunistas)** — PASS, con atención en Fase 2: el cambio de
  carrusel a cuadrícula toca el mismo componente (`pos-tables-panel.component.ts`) que ya sirve
  mesas y pedidos Domicilio/Para llevar (spec 059) — las tareas deben limitarse a lo que las 5
  historias de usuario piden, sin reescribir lógica no relacionada de ese componente.
- **Principio VI (Evolución incremental)** — PASS. Las 5 historias son independientes y
  priorizadas (P1 a P3); se implementan y verifican en unidades separadas.
- **Principio VII (Datos históricos)** — PASS, no aplica: no se toca ninguna venta ni factura ya
  emitida.
- **Principio VIII (Evolución del modelo de datos)** — PASS, no aplica: cero migraciones, cero
  columnas nuevas (ver Storage arriba); solo se expone un campo ya existente en un response ya
  existente.
- **Principio IX (Dependencias nuevas)** — PASS, no aplica: no se introduce ninguna dependencia
  nueva en ninguno de los dos repositorios.
- **Principio X (Verificación obligatoria)** — Pendiente de Fase 2 (tasks): cada historia de
  usuario necesita sus propios tests (frontend `*.spec.ts` nuevos/actualizados; backend, si el
  join de `user_id`→`user_name` requiere lógica nueva, un script `test_*.py` que la cubra).
  No bloquea el gate de este plan.
- **Principio XI (Negocio vs. técnico)** — PASS. Las decisiones de negocio (qué se implementa, con
  qué alcance) ya están tomadas en `spec.md`; este plan solo traduce eso a una estrategia técnica.
- **Principio XII (Trazabilidad)** — PASS. Cadena completa: Necesidad (imágenes de diseño) → Spec
  076 → Decisión (Clarifications 2026-09-04) → este Plan → Tasks (siguiente comando) → Tests.
- **Principio XIII (Español de Colombia)** — PASS. Este plan y los artefactos de Fase 0/1 se
  redactan en español de Colombia, igual que `spec.md`.

**Resultado del gate**: sin violaciones — no se requiere ninguna entrada en
`Complexity Tracking`.

## Project Structure

### Documentation (this feature)

```text
specs/076-terminal-mesas-rediseno-responsive/
├── plan.md              # This file (/speckit-plan command output)
├── research.md          # Phase 0 output (/speckit-plan command)
├── data-model.md        # Phase 1 output (/speckit-plan command)
├── quickstart.md        # Phase 1 output (/speckit-plan command)
├── contracts/           # Phase 1 output (/speckit-plan command)
│   └── order-response.md
└── tasks.md             # Phase 2 output (/speckit-tasks command - NOT created by /speckit-plan)
```

### Source Code (repositorios existentes, fuera de `pos-specs`)

Este proyecto de specs (`pos-specs`) no contiene código fuente: la implementación vive en los dos
repositorios en producción ya definidos por la Constitución. La estructura relevante ya existe —
esta feature no crea directorios nuevos, solo modifica/extiende los archivos listados:

```text
../pos-backend/                              # FastAPI + PostgreSQL (schema-per-tenant)
└── app/
    ├── models/
    │   └── customer_order.py                # user_id ya existe (línea 115) — sin cambios de modelo
    └── api/v1/orders/
        ├── schemas.py                        # OrderResponse (líneas 197-224) — agrega el campo expuesto
        └── service.py                        # serialización de la orden — agrega el join/lookup de usuario

../pos-heladeria/                             # Angular 21 (standalone, signals) + Tailwind 4
└── src/app/modules/
    ├── tables/
    │   ├── pages/
    │   │   └── table-sessions.component.ts       # barra superior nueva; contenedor del panel derecho unificado
    │   ├── components/
    │   │   ├── pos-tables-panel.component.ts      # carrusel → cuadrícula responsive; contadores de ocupación
    │   │   ├── order-summary-card.component.ts    # nuevo input "atendidoPor"; abreviaciones móviles
    │   │   └── pos-checkout-panel.component.ts     # placeholder "sin selección" con atajos (Historia 5)
    │   ├── services/
    │   │   └── pos-terminal.store.ts               # deriva contadores de ocupación; expone atendidoPor por mesa
    │   └── interfaces/
    │       └── dining.interface.ts                  # agrega el campo de usuario/atendido-por a DiningOrder
    └── cash-register/
        └── services/
            ├── cash-session.store.ts               # se consume (no se modifica) para leer/abrir turno
            └── cash.service.ts                      # openShift() ya existente, se invoca desde la nueva barra
```

**Structure Decision**: se reutiliza íntegramente la estructura de módulos ya existente en ambos
repositorios (patrón "Option 2: Web application" de este template, ya materializado como dos
repositorios separados en vez de dos carpetas hermanas). No se crea ningún módulo, servicio ni
directorio nuevo — todas las historias de usuario se resuelven extendiendo los archivos listados
arriba. El único archivo nuevo previsto es un componente pequeño para la barra superior operativa
(Historia 3), ubicado junto a los demás componentes de `tables/components/`.

## Constitution Check (post-diseño)

Re-evaluado tras Fase 1 (`research.md`, `data-model.md`, `contracts/`, `quickstart.md`): ningún
hallazgo de diseño introduce una entidad, columna, migración o dependencia nueva; el único campo
nuevo (`staff_user_name` en `OrderResponse`, ver `contracts/order-response.md`) es computado, igual
que `paid` ya lo es hoy en el mismo schema. El gate del Constitution Check arriba se mantiene sin
cambios — **sin violaciones**.

## Complexity Tracking

*Sin violaciones del Constitution Check — tabla no aplica.*
