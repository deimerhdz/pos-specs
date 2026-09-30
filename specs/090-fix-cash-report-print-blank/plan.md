# Implementation Plan: Reporte de Caja Legible al Imprimir (Cierre de Turno e Historial)

**Branch**: `090-fix-cash-report-print-blank` (rama de la spec en `pos-specs`; la rama de código de `pos-heladeria` sigue el Principio XIV, ver abajo) | **Date**: 2026-09-30 | **Spec**: [spec.md](./spec.md)

**Input**: Feature specification from `/specs/090-fix-cash-report-print-blank/spec.md`

## Summary

Al imprimir el reporte de caja (tras cerrar el turno o desde el historial) en Brave, la hoja sale con solo el encabezado y pie del navegador. La spec 089 (Historia 5) lo declaró corregido tras una verificación parcial (Chrome headless sin fondos) y el síntoma persiste. Todo lo siguiente sale de leer `pos-heladeria` (`develop`, árbol limpio), la rama `fix/089-cash-report-print` y las specs 087/089:

1. **Hecho clave que la spec solo sospechaba**: el arreglo de 089 **no está en `develop`**. `fix/089-cash-report-print` nunca se fusionó (el PR #93 trajo solo adicionales, cierre de mesa y total del POS). Por eso el plan no confía en ninguna causa: la **primera tarea es el diagnóstico en Brave** (FR-009) sobre **dos estados** (`develop` a secas y `develop` + la rama de 089) y con hipótesis H1–H5 a descartar con evidencia (código viejo/service worker, vista previa real ≠ `printToPDF`, oscurecimiento forzado del navegador o de una extensión, tema del SO, capas `fixed` del shell). La decisión posterior sigue una tabla de reglas (research D1).
2. **Diseño de la corrección**: dejar de imprimir **la página de la app** y pasar a imprimir el reporte como **documento HTML independiente en un iframe oculto** (el patrón ya vigente de los recibos), con estilos propios negro sobre blanco, `color-scheme: only light`, `print-color-adjust: exact`, sin fondos ni colores de estado y sin recursos externos. Es la única solución que elimina **por construcción** las causas posibles —tema, fondos, capas del shell, alturas, `overflow`— en vez de combatirlas con `!important`, que es lo que ya falló (research D2–D4). El contenido y las cifras no cambian: salen de las mismas señales del store.
3. **Nada de backend ni de datos.** Solo `pos-heladeria`: un módulo nuevo de TypeScript puro (`cash-report-print.util.ts`: generador del documento + impresión en iframe), `imprimirReporte()` del store adaptado, y retiro del mecanismo `body.printing-cash-report` de spec 087 (queda sin efecto). `receipt.util.ts` y la hoja de QR **no se tocan**.
4. **Verificación real (FR-010)**: vista previa y PDF guardado **en Brave**, ambas rutas, 4 combinaciones de tema × fondos; y una **comprobación automática repetible** (`npm run verify:cash-report-print`) que imprime el documento a PDF con Playwright emulando `print`, sin depender de fondos, y confirma **texto presente y tinta visible en cada hoja**, con un control negativo (texto blanco) que debe fallar. Leer solo el texto del PDF no basta: el texto blanco también se extrae.
5. **Registro (FR-011)**: A-98 ya está en el registro; se amplía con la causa confirmada y la Historia 5 de 089 queda marcada como reemplazada por esta spec.

Sin dependencias nuevas (Playwright ya es `devDependency`; poppler es herramienta del sistema). Sin migración. Sin cambio de contrato.

## Technical Context

**Language/Version**: Angular 21 / TypeScript (`pos-heladeria`). `pos-backend` (Python 3.12) **sin cambio**.

**Primary Dependencies**: Angular signals y Tailwind en la app; **ninguna dependencia nueva** (Principio IX). El generador del documento no usa Tailwind ni Angular a propósito. Herramientas de verificación ya presentes: `playwright` (devDependency declarada, hoy sin uso en el repo), `typescript` (para cargar el módulo puro desde el script), `pdftotext`/`pdftoppm` del sistema.

**Storage**: N/A — no hay cambio de esquema ni de datos. Ver [data-model.md](./data-model.md) (solo un modelo de presentación en memoria).

**Testing**: `ng test` (`@angular/build:unit-test` con Vitest y jsdom, como el resto del repo) para el generador, el store y el componente (ningún test unitario ejecuta impresión real); `scripts/verify-cash-report-print.mjs` (Playwright + poppler) para PDF; verificación **manual en Brave** (vista previa real y PDF guardado) documentada en [quickstart.md](./quickstart.md).

**Target Platform**: navegador de escritorio **Brave** (Chromium) con vista previa de impresión y "Guardar como PDF" (obligatorio); Chrome equivalente. Firefox, Safari, móviles e impresoras térmicas fuera de alcance.

**Project Type**: web — `pos-backend` + `pos-heladeria`; aquí solo cambia `pos-heladeria`; la spec vive en `pos-specs`.

**Performance Goals**: el diálogo de impresión abre en < 1 s tras pulsar (documento pequeño, sin recursos externos); un reporte de 70 movimientos se genera y pagina sin bloqueo perceptible. SC-006: < 30 s desde ver el reporte hasta tener el PDF guardado.

**Constraints**:
- **FR-008 / SC-005**: recibo de venta, recibo de mesa y hoja de QR **no cambian** (`receipt.util.ts` y `table-qr-sheet.component.ts` intactos; se comprueba con `git diff` vacío).
- **Independiente de tema, fondos y tamaño** (FR-003/004/005): ningún texto depende de un fondo; negro puro sobre blanco.
- **Seguridad**: todo texto de usuario/base se escapa al armar el HTML a mano (research D9).
- **Nombre de archivo** `<negocio>-<DD-MM-YYYY>` con fecha de **cierre** (spec 087) se conserva (research D5, con verificación en Brave).
- **Diagnóstico antes de corregir** (FR-009): no se escribe código de la corrección sin el registro en `implementation-notes.md`.

**Scale/Scope**:
- `pos-heladeria` (`src/app/modules/…`): `cash-register/services/cash-report-print.util.ts` (**nuevo**, + `.spec.ts`), `cash-register/services/cash-session.store.ts` (`imprimirReporte()`, + spec), `cash-register/components/cash-report.component.ts` (retirar su `@media print`), `dashboard/layout/dashboard-layout.component.ts` (retirar reglas `printing-cash-report`); `scripts/verify-cash-report-print.mjs` (**nuevo**) y una línea en `package.json`.
- `pos-specs`: `implementation-notes.md` (diagnóstico y verificación), A-98 ampliada, nota de reemplazo en la spec 089.
- 3 historias: P1×2 (cierre e historial; misma pantalla y misma acción) · P2 (garantía transversal y verificación real).

## Constitution Check

*GATE: debe pasar antes de la Fase 0. Re-evaluado tras la Fase 1 (final de la sección).*

| Principio | Evaluación |
|---|---|
| I. Las Nuevas Funcionalidades Nacen de un Spec | ✅ Pass — `spec.md` con clarificación (2026-09-30), checklist de calidad y 11 FR. |
| II. El Comportamiento Existente Sigue Protegido | ✅ Pass — no cambia ningún comportamiento de negocio: cifras, secciones, nombre de archivo, acción visible y ocultamiento de menú/encabezado/botones se **conservan**. Cambia solo el mecanismo de impresión y, en papel, se neutralizan colores de estado (ya decidido en 089 y registrado en **A-98**). |
| III. Los Characterization Tests Protegen el Comportamiento Heredado | ✅ Pass — ningún `CONGELA` se toca (no hay cambio de backend). Verificado en `develop`: ninguna prueba de frontend congela `printing-cash-report` ni `imprimirReporte()` (solo un `vi.fn()` de `cash-report.component.spec.ts`), así que retirar el mecanismo de 087 no contradice pruebas existentes; si alguna apareciera al implementar, se actualiza **citando A-98** en el mismo commit. Evidencia: suite completa en verde y `git diff` vacío en recibos y hoja de QR. |
| IV. Los Nuevos Specs Pueden Introducir Nuevo Comportamiento | ✅ Pass — comportamiento acotado y medible (SC-001…SC-006). |
| V. Nuevas Funcionalidades Antes que Refactorizaciones Oportunistas | ✅ Pass — no se refactoriza `receipt.util.ts` ni se extrae una utilidad compartida (research D3): se acepta ~30 líneas duplicadas para no tocar la impresión de recibos. El retiro de `printing-cash-report` no es oportunista: es la otra mitad del mecanismo que se reemplaza (D6). No se integra `fix/089-cash-report-print`. |
| VI. Evolución Incremental | ✅ Pass — una sola clase de cambio (presentación de impresión en el cliente), en unidades verificables: (1) diagnóstico documentado; (2) generador puro + pruebas; (3) impresión en iframe + store; (4) retiro del mecanismo antiguo; (5) comprobación automática; (6) verificación manual en Brave. Sin mezclar backend, datos ni refactors. |
| VII. Compatibilidad con Datos Históricos | ✅ Pass — no se lee ni escribe ningún dato distinto; turnos cerrados, movimientos y arqueos históricos se imprimen con las mismas cifras (las reimpresiones del historial usan el mismo generador). Ninguna factura se toca. |
| VIII. Evolución del Modelo de Datos | ✅ Pass (no aplica) — [data-model.md](./data-model.md) declara explícitamente **cero cambios de esquema o datos**; la reversión es revertir commits de frontend. |
| IX. Dependencias Nuevas Permitidas con Justificación | ✅ Pass (no aplica) — cero dependencias nuevas. Generar PDF en cliente (jsPDF/pdfmake) y en servidor se evaluaron y **descartaron** (research D2, alternativas 3–4). |
| X. Verificación Obligatoria | ✅ Pass (planificado) — pruebas de unidad + comprobación automática a PDF (con control negativo) + verificación **manual en Brave** con vista previa real y PDF guardado, ambas rutas y 4 combinaciones (FR-010); el diagnóstico real **precede** a la corrección (FR-009). Es justo la brecha de verificación que dejó abierta 089 (T070). |
| XI. Decisiones de Negocio Frente a Decisiones Técnicas | ✅ Pass — qué debe verse (negro sobre blanco, completo, legible) es la decisión de negocio ya en la spec; el plan solo decide el "cómo" (documento aislado). |
| XII. Trazabilidad | ✅ Pass — Necesidad (capturas del usuario, 2026-09-30) → spec + clarificación → A-98 → este plan → research/data-model/contracts/quickstart → `tasks.md` → tests y verificación en Brave. |
| XIII. Todo en Español de Colombia | ✅ Pass — artefactos, comentarios y textos del reporte impreso en español de Colombia (mismos textos que la pantalla). |
| XIV. Estrategia y Convención de Ramas | ✅ Pass (planificado) — antes de tocar código se crea en `pos-heladeria`, **desde `develop`** (la rama actual; no desde `fix/089-cash-report-print`), `fix/090-cash-report-print-isolated`. `pos-backend` no se toca, no lleva rama. `/speckit-tasks` lo deja como tarea inicial. |
| XV. Política de Commits | ✅ Pass (planificado) — commits pequeños (generador + pruebas; impresión en iframe; store; retiro del mecanismo antiguo; script de verificación; notas/spec), en inglés, Conventional Commits, **sin marcas de IA**, solo cuando el usuario lo pida. |

**Complexity Tracking**: sin violaciones que justificar. La única "duplicación" (iframe de impresión) está justificada en research D3 y no es una violación de principio.

**Re-chequeo post Fase 1**: el diseño no añadió dependencias, variables de entorno ni contratos HTTP. Dos hallazgos quedaron **absorbidos** sin cambiar el veredicto: (1) el generador debe ser **TS puro sin imports de Angular** para que el script de verificación lo cargue con `typescript` (sin bundler ni dependencia nueva); (2) al armar HTML a mano se pierde el escape automático de Angular ⇒ escapador explícito con prueba propia (D9). **Riesgo abierto, declarado y con verificación asignada**: qué título usa Brave como nombre sugerido al imprimir desde un iframe (D5); se fijan ambos títulos y se comprueba en la vista previa real; si difiere, se ajusta antes de cerrar. **Condición de la regla de decisión** (D1): si el diagnóstico prueba que la causa era solo que el arreglo de 089 nunca llegó y `develop`+089 pasa las 4 combinaciones en Brave, el alcance se reduce a integrarlo y verificarlo, como anticipa la spec (Suposiciones); esa rama de decisión la toma el diagnóstico, no este plan.

## Project Structure

### Documentation (this feature)

```text
specs/090-fix-cash-report-print-blank/
├── spec.md              # Especificación (con clarificación 2026-09-30)
├── plan.md              # Este archivo (/speckit-plan)
├── research.md          # Fase 0: hechos verificados + decisiones D1–D11
├── data-model.md        # Fase 1: sin cambio de datos; modelo de presentación en memoria
├── quickstart.md        # Fase 1: diagnóstico, comprobación automática y verificación manual en Brave
├── contracts/
│   └── cash-report-print.md   # Fase 1: interfaz, invariantes I-1…I-12 y ciclo de impresión
├── checklists/
│   └── requirements.md
├── implementation-notes.md    # (se crea al implementar) diagnóstico, decisiones y verificación
└── tasks.md             # Fase 2 (/speckit-tasks — NO lo crea /speckit-plan)
```

### Source Code (repository root)

Solo cambia `../pos-heladeria`. `../pos-backend` **sin cambios**.

```text
../pos-heladeria/
├── src/app/modules/
│   ├── cash-register/
│   │   ├── services/
│   │   │   ├── cash-report-print.util.ts        # NUEVO — TS puro: CashReportPrintData, buildCashReportHtml, printCashReportHtml
│   │   │   ├── cash-report-print.util.spec.ts   # NUEVO — secciones, escape, sin Tailwind/externos, título, invariantes I-1…I-12
│   │   │   ├── cash-session.store.ts            # imprimirReporte(): arma datos y llama al util; sin body.classList
│   │   │   └── cash-session.store.spec.ts       # título por fecha de cierre, slug de respaldo, sin clases en body
│   │   └── components/
│   │       └── cash-report.component.ts         # se retira su @media print; botones conservan print:hidden
│   ├── dashboard/layout/
│   │   └── dashboard-layout.component.ts        # se retiran las reglas :host-context(body.printing-cash-report) (spec 087)
│   └── tables/                                  # SIN CAMBIOS: receipt.util.ts, table-qr-sheet.component.ts
├── scripts/
│   └── verify-cash-report-print.mjs             # NUEVO — Playwright + poppler, 4 combinaciones, control negativo
└── package.json                                 # + "verify:cash-report-print"
```

**Structure Decision**: web con un solo repositorio afectado (`pos-heladeria`). El generador del documento vive en `cash-register/services/` como módulo **sin dependencias de Angular** (research D7) junto al store que lo consume, en vez de en `shared/`: hoy solo lo usa el reporte de caja y promoverlo a compartido sería una abstracción prematura (Principio V).

## Complexity Tracking

> Sin violaciones de la Constitución que justificar (ver Constitution Check).
