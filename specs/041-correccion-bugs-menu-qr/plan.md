# Implementation Plan: Corrección de bugs y mejoras — Menú QR

**Branch**: `041-correccion-bugs-menu-qr` | **Date**: 2026-08-27 | **Spec**: [spec.md](./spec.md)

**Input**: Feature specification from `/specs/041-correccion-bugs-menu-qr/spec.md`

## Summary

Cuatro correcciones/mejoras independientes, todas dentro de `pos-heladeria` (Angular), sin ningún
cambio en `pos-backend`:

1. **Bug 1** — `PublicMenuComponent.exit()` (`public-menu.component.ts:1013-1032`) hoy invalida
   correctamente el `session_token` (servidor y `localStorage`), pero solo cambia el `view` local a
   `'error'` sin dejar ninguna marca que sobreviva a un reload. `ngOnInit`
   (`public-menu.component.ts:651-689`) siempre evalúa `tokenStore.token()` de cero y, si está
   vacío, muestra `view.set('name')` (línea 677) — el mismo estado que un primer acceso legítimo,
   permitiendo crear una sesión nueva sin haber vuelto a escanear el QR. La corrección agrega una
   marca de "acceso cerrado" del lado del navegador (Clarifications, sesión 2026-08-27: la URL no
   debe funcionar como credencial por sí sola) que `ngOnInit` consulta antes de decidir el `view`.
2. **Bug 2** — `qrDataUrl`/`buildTableQr` (`table-qr.util.ts:11-34`) generan el PNG del QR desnudo,
   sin texto, con un único tamaño. La corrección compone, sobre un `<canvas>` nativo, el QR ya
   generado por `qrcode` junto con el identificador de la mesa (`DiningTable.number`/`name`), en dos
   variantes de tamaño ("Mostrador"/"Sticker"), reutilizadas tanto en la descarga individual
   (`table-qr.component.ts`) como en la generación masiva (`table-qr-sheet.component.ts`).
3. **Bug 3** — `public-menu.component.ts:346-350` renderiza el emoji `🍦` cuando `product.image_url`
   es nulo. La corrección agrega un caso nuevo `"image-off"` al switch de íconos ya existente
   (`shared/icon/icon.component.ts`) y lo usa ahí en vez del emoji.
4. **Bug 4** — el botón "Copiar insumos..." (`product-form.component.ts:384-389`) hoy solo depende
   de `draft().hasSizes && draft().variants.length > 1`. La corrección agrega
   `draft().tracks_inventory` a esa misma condición del template — ningún cambio de lógica adicional
   porque la operación es puramente en memoria (`copyConfigToOthers`, sin endpoint propio, FR-026).

Ningún bug requiere migración de base de datos, endpoint nuevo, ni dependencia nueva.

## Technical Context

**Language/Version**: TypeScript 5.9.2 / Angular 21.1.0 (`pos-heladeria`). `pos-backend` (Python /
FastAPI 0.136) no se toca en esta spec — confirmado por FR-007, FR-012 y FR-026 de `spec.md`.

**Primary Dependencies**: Angular standalone components + signals (patrón ya en uso, sin cambios de
versión); `qrcode` (`^1.5.4`, `QRCode.toDataURL`, ya en uso en `table-qr.util.ts`) — Bug 2 lo
reutiliza sin agregarlo de nuevo, componiendo el resultado sobre la API `Canvas 2D` nativa del
navegador (sin librería nueva) para dibujar el identificador de mesa; `shared/icon/icon.component.ts`
(switch de íconos SVG propio, sin librería de terceros) — Bug 3 le agrega un caso nuevo. Ningún
import nuevo en ningún bug (Principio IX: no aplica justificación porque no se añade nada).

**Storage**: PostgreSQL 16 schema-per-tenant en producción — **sin cambios de esquema**; ningún bug
de esta spec agrega columna, tabla ni migración. El único almacenamiento nuevo es del lado del
navegador: una marca de "acceso cerrado" para Bug 1 (mecanismo exacto decidido en Phase 0,
`research.md`, Decisión 1 — es un detalle técnico, no de negocio, según Clarifications de `spec.md`).

**Testing**: Frontend — `@angular/build:unit-test` sobre Vitest (`pos-heladeria/package.json`,
`"vitest": "^4.0.8"`). Hoy **no existe** ningún `*.spec.ts` para `public-menu.component.ts`,
`table-qr.component.ts`, `table-qr.util.ts`, `table-qr-sheet.component.ts` ni
`diner-token.store.ts` — esta spec crea el primer test de cada uno de los archivos que toca (Bug 1 y
Bug 2). `product-form.component.spec.ts` ya existe (Bug 4) — se extiende, no se crea. Backend — sin
cambios, por lo tanto sin tests nuevos de `pos-backend` (`python -m unittest`, patrón existente, sin
`pytest.ini` en el repo).

**Target Platform**: navegador móvil del comensal (Menú QR — Bug 1 y Bug 3) y navegador de
escritorio/tablet del panel de administración (Bug 2 y Bug 4); ambos dentro de la misma SPA Angular
de `pos-heladeria`, sin app nativa ni build separado.

**Project Type**: corrección/mejora acotada al frontend único de `pos-heladeria` — ningún archivo de
`pos-backend` cambia en esta spec (a diferencia del patrón "web application" de dos proyectos, aquí
solo un lado del sistema se modifica).

**Performance Goals**: sin objetivo nuevo. Bug 2 agrega una composición de `canvas` en el momento de
la descarga (operación local, una sola vez por clic, sin llamada de red adicional a las ya
existentes). Los demás bugs son cambios de condición/render sin costo de cómputo relevante.

**Constraints**: no se modifica ningún endpoint de `pos-backend` (FR-007, FR-026); no se modifica la
información codificada en el QR (FR-012); no se toca la regla protegida A-24/`RN-CART-24`
(permanencia del token QR, spec 007) ni la expiración por inactividad (`RN-CART-17` a `RN-CART-20`);
no se toca el placeholder de las pantallas de staff/admin que reutilizan el mismo emoji (FR-020); no
se modifica ninguna regla ya vigente de manejo de inventario/receta/insumos (FR-025, spec 003/027).

**Scale/Scope**: 4 correcciones independientes, cada una acotada a un puñado de archivos ya
existentes dentro de `pos-heladeria/src/app/modules/tables/**` (Bug 1 y 2),
`pos-heladeria/src/app/modules/products/**` (Bug 4) y `pos-heladeria/src/app/shared/icon/**` (Bug 3);
ningún archivo de `pos-backend`; sin migraciones.

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| Principio | Evaluación | Estado |
|---|---|---|
| **I. Las Nuevas Funcionalidades Nacen de un Spec** | `spec.md` (041) existe, aprobado y clarificado, antes de este plan y de cualquier cambio de código. | PASS |
| **II. El Comportamiento Existente Sigue Protegido** | Único cambio de comportamiento observable existente: Bug 1 (hoy reload/back/forward tras logout permite crear sesión nueva; esta spec lo bloquea). Documentado con quién (dueño/desarrollador del proyecto), cuándo (2026-08-27), qué cambia (invalidación de acceso tras logout explícito), por qué (brecha de seguridad/integridad de sesión) y qué se ve afectado (`PublicMenuComponent`, `DinerTokenStore`) — en `spec.md` §"Autorización de negocio" y §Clarifications. Bug 2/3/4 no cambian ninguna regla de negocio existente, solo presentación/visibilidad de UI. | PASS |
| **III. Los Characterization Tests Protegen el Comportamiento Heredado** | No existe hoy ningún test `"CONGELA comportamiento actual:"` sobre ninguno de los 6 archivos que esta spec toca (ni `pos-heladeria` los usa como convención — esa convención es de `pos-backend/app/characterization_tests/`). Ningún test protegido de `pos-backend` se ve afectado porque `pos-backend` no cambia. | PASS |
| **IV. Los Nuevos Specs Pueden Introducir Nuevo Comportamiento** | Bug 1 introduce comportamiento nuevo (bloqueo de reingreso) de forma explícita y aprobada por `spec.md`. | PASS |
| **V. Nuevas Funcionalidades Antes que Refactorizaciones Oportunistas** | Alcance estrictamente acotado a los archivos citados en `spec.md`; el §Out of Scope excluye explícitamente tocar las pantallas de staff/admin que comparten el mismo emoji (Bug 3), cualquier regla de inventario existente (Bug 4), y cualquier endpoint de `pos-backend` (Bug 1/2). Ningún refactor no relacionado viaja en este cambio. | PASS |
| **VI. Evolución Incremental** | Los 4 bugs son la misma clase de cambio (corrección/mejora de UI y de invalidación de sesión, sin migración ni cambio de arquitectura) y no comparten código entre sí (Bug 1: `public-menu`/`diner-token`; Bug 2: `table-qr*`; Bug 3: `public-menu` + `icon.component`; Bug 4: `product-form`) — cada uno tiene su propia User Story independientemente verificable en `spec.md`, y `tasks.md` (siguiente fase) organiza el trabajo por historia para que cada corrección pueda entregarse y verificarse por separado dentro de este mismo incremento agrupado (el propio reporte de bugs del dueño del producto los agrupó explícitamente en una sola spec). | PASS |
| **VII. Compatibilidad con Datos Históricos** | No aplica — ningún bug toca facturación, importes ni datos históricos. | PASS (no aplica) |
| **VIII. Evolución del Modelo de Datos** | No aplica — sin cambios al esquema de `pos-backend`. La única entidad nueva es un dato exclusivamente de navegador (marca de "acceso cerrado", Bug 1), documentada en `data-model.md` por trazabilidad aunque no sea una migración. | PASS (no aplica) |
| **IX. Dependencias Nuevas Permitidas con Justificación** | No aplica — ningún bug agrega una dependencia nueva; Bug 2 reutiliza `qrcode` (ya en uso) + `Canvas 2D` nativo; Bug 3 reutiliza el switch de íconos propio ya existente. | PASS (no aplica) |
| **X. Verificación Obligatoria** | Se crean tests nuevos para los 5 archivos sin `*.spec.ts` hoy (Bug 1 y 2) y se extiende `product-form.component.spec.ts` (Bug 4); `quickstart.md` documenta la verificación manual de los pasos que un test unitario no puede cubrir por sí solo (escaneo real con cámara, impresión). | PASS |
| **XI. Decisiones de Negocio Frente a Decisiones Técnicas** | `spec.md` §Clarifications separa explícitamente la decisión de negocio (rechazar reingreso vía URL/historial/reload/reapertura, sin vencimiento automático) de la decisión técnica que resuelve este plan (mecanismo exacto de la marca de "acceso cerrado", `research.md` Decisión 1). | PASS |
| **XII. Trazabilidad** | Cadena completa: reporte de bugs del dueño del proyecto → `spec.md` (041) + Clarifications → este plan (`research.md`, `data-model.md`, `contracts/`) → implementación (`tasks.md`, siguiente fase) → tests nuevos → `quickstart.md`. | PASS |
| **XIII. Todo en Español de Colombia** | `spec.md`, este plan y los artefactos que genera (`research.md`, `data-model.md`, `contracts/`, `quickstart.md`) están en español de Colombia. | PASS |

Sin violaciones. La tabla de Complexity Tracking al final de este documento queda vacía.

**Re-chequeo posterior a Phase 1 (research.md/data-model.md/contracts/quickstart.md)**: ninguna
decisión de diseño (marca de `sessionStorage` para Bug 1, composición por `canvas` para Bug 2,
caso nuevo de ícono para Bug 3, condición de template para Bug 4) introduce dependencia nueva,
cambio de esquema, ni contrato de backend nuevo — la tabla anterior permanece sin cambios tras el
diseño.

## Project Structure

### Documentation (this feature)

```text
specs/041-correccion-bugs-menu-qr/
├── plan.md              # Este archivo (/speckit-plan)
├── research.md          # Fase 0 (/speckit-plan)
├── data-model.md         # Fase 1 (/speckit-plan)
├── quickstart.md         # Fase 1 (/speckit-plan)
├── contracts/             # Fase 1 (/speckit-plan) — sin contratos nuevos, ver contracts/README.md
└── tasks.md              # Fase 2 (/speckit-tasks — NO se crea en este comando)
```

### Source Code (repositorio)

```text
pos-heladeria/src/app/
├── modules/
│   ├── tables/
│   │   ├── pages/
│   │   │   ├── public-menu.component.ts            # Bug 1 (marca de acceso cerrado) + Bug 3 (ícono)
│   │   │   ├── public-menu.component.spec.ts        # NUEVO — Bug 1 y Bug 3
│   │   │   └── table-qr-sheet.component.ts          # Bug 2 — generación masiva, variantes de tamaño
│   │   ├── components/
│   │   │   ├── table-qr.component.ts                 # Bug 2 — descarga individual, selector Mostrador/Sticker
│   │   │   └── table-qr.component.spec.ts             # NUEVO — Bug 2
│   │   └── services/
│   │       ├── table-qr.util.ts                       # Bug 2 — composición del PNG con identificador de mesa
│   │       ├── table-qr.util.spec.ts                   # NUEVO — Bug 2
│   │       ├── diner-token.store.ts                    # Bug 1 — marca de "acceso cerrado"
│   │       └── diner-token.store.spec.ts                # NUEVO — Bug 1
│   └── products/
│       └── pages/
│           ├── product-form.component.ts               # Bug 4 — condición del botón "Copiar insumos..."
│           └── product-form.component.spec.ts            # YA EXISTE — se extiende, Bug 4
└── shared/
    └── icon/
        └── icon.component.ts                            # Bug 3 — nuevo caso "image-off"

pos-backend/                                             # SIN CAMBIOS en esta spec
```

**Structure Decision**: corrección de un único proyecto (`pos-heladeria`), sin tocar
`pos-backend`. No aplica ninguna de las opciones genéricas de proyecto múltiple del template — se
documentan directamente las rutas reales dentro del árbol ya existente de `pos-heladeria`.

## Complexity Tracking

*Sin violaciones que justificar — tabla vacía.*
