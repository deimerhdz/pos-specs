---

description: "Tareas de la spec 090 — reporte de caja legible al imprimir (cierre de turno e historial), impresión en documento aislado"
---

# Tasks: Reporte de Caja Legible al Imprimir (Cierre de Turno e Historial)

**Input**: Documentos de diseño de `/specs/090-fix-cash-report-print-blank/`

**Prerequisites**: [plan.md](./plan.md), [spec.md](./spec.md), [research.md](./research.md) (D1–D11), [data-model.md](./data-model.md), [contracts/cash-report-print.md](./contracts/cash-report-print.md) (invariantes I-1…I-12), [quickstart.md](./quickstart.md)

**Tests**: La spec y el plan (research D7/D8) piden pruebas de unidad (`ng test` = `@angular/build:unit-test` con **Vitest + jsdom**, `vi.fn()`; no Karma/Jasmine), una comprobación automática a PDF con control negativo y la verificación manual en Brave. Cada tarea de implementación va precedida de su prueba (se escribe primero y debe fallar). **Ninguna prueba unitaria ejecuta impresión real**: la impresión solo se da por verificada con la vista previa y el PDF guardado en Brave (FR-010; es la brecha que dejó abierta la spec 089, T070).

**Organización**: por historia de usuario en el orden de prioridad de `spec.md` (US1 P1 → US2 P1 → US3 P2). US1 y US2 comparten **la misma pantalla y la misma acción** (`imprimirReporte()`); la implementación del reporte aislado vive en US1 y US2 la comprueba y cubre lo propio de la ruta del historial.

## Formato: `[ID] [P?] [Story] Descripción`

- **[P]**: se puede ejecutar en paralelo (archivo distinto, sin dependencia de una tarea sin terminar)
- **[Story]**: historia de usuario (US1–US3)
- Rutas relativas a `../pos-heladeria` (`src/app/modules/…`) o `pos-specs` (esta carpeta) según se indique. Abreviatura: `cash/` = `../pos-heladeria/src/app/modules/cash-register/`

## Notas de gobernanza y ramas

- **`pos-backend` no se toca** (sin rama, sin migración, sin cambio de contrato). Solo cambia `pos-heladeria` (+ documentación en `pos-specs`).
- **Rama (Principio XIV)**: `fix/090-cash-report-print-isolated` en `../pos-heladeria`, **desde `develop`** (no desde `fix/089-cash-report-print`, que **no se integra**; research D6/D10).
- **FR-009 manda**: el diagnóstico en Brave (Fase 2) se registra en `implementation-notes.md` **antes del primer commit de código**. **Regla de decisión (research D1)**: si el diagnóstico cae en la fila "basta integrar 089" o en "causa distinta", se **detiene** la Fase 3 y se revisa `research.md`/este archivo antes de continuar.
- **Commits (Principio XV)** solo cuando el usuario lo pida: pequeños, en inglés, Conventional Commits, sin marcas de IA. Agrupación sugerida: (1) generador + pruebas; (2) impresión en iframe + pruebas; (3) store; (4) retiro del mecanismo antiguo; (5) script de verificación; (6) notas/A-98/nota en 089 (en `pos-specs`).
- **Fuera de alcance**: `receipt.util.ts`, `table-qr-sheet.component.ts`, backend, contenido y cifras del reporte, nombre de archivo (se conserva), Firefox/Safari/móviles/térmicas (research D11).

---

## Phase 1: Setup

**Propósito**: rama y prerrequisitos de verificación.

- [X] T001 Crear en `../pos-heladeria` la rama `fix/090-cash-report-print-isolated` **desde `develop`** (árbol limpio; hoy está en `develop`) y confirmar con `git merge-base --is-ancestor fix/089-cash-report-print HEAD` que la rama de 089 **no** está contenida (debe devolver código ≠ 0)
- [X] T002 [P] Comprobar los prerrequisitos del quickstart §0 y anotarlos en `implementation-notes.md` (archivo NUEVO en esta carpeta, con las secciones "Diagnóstico", "Decisiones", "Verificación automática", "Verificación manual en Brave" y "Limitaciones"): `/usr/bin/brave-browser` y una ruta para Chrome, `pdftotext`/`pdftoppm` (`poppler-utils`), `playwright` resolvible con `npx playwright --version` desde `../pos-heladeria`, y que `npx ng test --watch=false` corre en verde **antes** de cambiar nada (línea base)
- [ ] T003 Preparar datos de prueba en un negocio de **desarrollo** (nunca producción; quickstart §0): un turno cerrado con movimientos (ingresos, egresos, retiros y ventas por más de un método de pago), uno cerrado **sin movimientos**, uno cerrado de un **día anterior** y uno con **60+ movimientos**; dejar anotado en `implementation-notes.md` cómo se armaron (script o pasos). Además, **antes de cambiar código**, en `develop` imprimir en Brave (vista previa y PDF guardado) un recibo de venta, un recibo de mesa y la hoja de QR y guardar capturas y PDF en `implementation-notes.md` (sección "Verificación manual en Brave", subsección "Línea base FR-008"); T023 los usa como referencia

---

## Phase 2: Foundational — Diagnóstico en Brave (FR-009) ⚠️ BLOQUEA todo

**Propósito**: reproducir el defecto en la vista previa **real** y registrar la causa antes de escribir la corrección (research D1). Sin este registro no se inicia la Fase 3.

**Checkpoint**: la Fase 3 solo arranca cuando T007 deja una fila de la "Regla de decisión" anotada y fechada.

- [ ] T004 Diagnóstico **E0** (`develop` sin arreglo): levantar `ng serve` en `../pos-heladeria`, abrir el reporte de un turno cerrado con movimientos en Brave con **perfil limpio** (`brave-browser --user-data-dir=$(mktemp -d)`), pulsar "Imprimir / Exportar reporte" y registrar en `implementation-notes.md` (sección "Diagnóstico") con captura de la **vista previa real**: con "Gráficos de fondo" desactivado y activado, y el **PDF guardado** (quickstart §1 pasos 1–3)
- [ ] T005 Diagnóstico **E1** (`develop` + `fix/089-cash-report-print`): en una **rama desechable** de `../pos-heladeria` creada desde `develop` (p. ej. `chore/090-diag-e1`, se descarta al terminar; **no** tocar la rama de trabajo de T001) fusionar `fix/089-cash-report-print` y repetir exactamente T004; registrar resultados comparados con E0
- [ ] T006 Descartar hipótesis **H1–H5** de research D1 con evidencia en `implementation-notes.md`: (H1) versión del bundle en el navegador del usuario frente a `develop` (Application → Service Workers, hash de `main-*.js`) y **perfil del usuario** vs perfil limpio; (H2) vista previa real vs `printToPDF` emulado en E1, con DevTools → Rendering → `Emulate CSS media type: print`; (H3) oscurecimiento forzado del navegador/extensión (perfil limpio vs usuario; `Emulation.setAutoDarkModeOverride`); (H4) `prefers-color-scheme: dark` del SO con los colores **calculados** (`color`, `background-color`, `print-color-adjust`) del texto y de sus ancestros; (H5) capas `fixed` (toast, `confirm-dialog`, telón) por bisección. Para cada una: confirmada / descartada, captura, PDF y perfil usado
- [ ] T007 Cerrar el diagnóstico: en `implementation-notes.md` anotar (con fecha) **qué fila de la regla de decisión de research D1 aplica**, por qué el arreglo de 089 no bastó o no llegó (insumo de FR-011/A-98), y descartar la rama temporal de T005. **Si la fila es "basta integrar 089" o "causa distinta": PARAR y revisar `research.md` y este `tasks.md` con el usuario antes de la Fase 3**

---

## Phase 3: User Story 1 — Imprimir el reporte justo después de cerrar el turno (Priority: P1) 🎯 MVP

**Goal**: "Imprimir / Exportar reporte" tras cerrar el turno imprime un **documento HTML aislado** (iframe oculto, negro sobre blanco, sin shell) con todo el contenido legible y el nombre `<negocio>-<DD-MM-YYYY>`.

**Independent Test**: cerrar un turno con movimientos en Brave, pulsar "Imprimir / Exportar reporte": la vista previa y el PDF guardado muestran todos los bloques legibles; turno sin movimientos imprime encabezados y totales en cero (quickstart §3, Ruta A).

### Generador del documento (contrato §1–§3)

- [ ] T008 [US1] Escribir primero `cash/services/cash-report-print.util.spec.ts` (NUEVO; Vitest + jsdom) para `buildCashReportHtml`: (a) presencia y **orden** de secciones (encabezado: negocio, "Reporte de cierre — caja", cajero, apertura, cierre → "Resumen financiero" → "Arqueo de caja" + observación solo si hay → "Movimientos del turno (N)") (I-5); (b) turno **sin movimientos** ⇒ "No se registraron movimientos en este turno." y totales presentes (I-6); (c) `cambioEntregado` `null` ⇒ fila omitida; (d) **escape de HTML** de negocio, cajero, caja, categoría/nota de movimientos, observación y nombres de método de pago, con caso `<img src=x onerror=alert(1)>` impreso literal y sin `<script>` ni `on*=` en el resultado (I-10, research D9); (e) contiene `color-scheme: only light` (CSS y `<meta>`), `print-color-adjust: exact`, `forced-color-adjust: none`, `@page { size: A4; margin: 10mm }`, `thead` con `display: table-header-group` y `break-inside: avoid`, sin `height` fija ni `overflow` (I-2, I-7, I-8); (f) **sin** atributos `class=`, sin `<link>`, `<script>`, `src=` ni `url(` externos (I-9); (g) `<title>` = `fileTitle` (I-11); (h) ningún color de estado ni fondo distinto de `#fff` (I-1, I-3, I-4); (i) la diferencia se lee por texto (Cuadre perfecto/Sobrante/Faltante) (I-4); (j) determinismo: mismo dato ⇒ mismo HTML. Ejecutar y ver que falla
- [ ] T009 [US1] Crear `cash/services/cash-report-print.util.ts` (NUEVO) **sin importar nada de Angular** (lo carga el script de la Fase 5 con `typescript`): exportar `CashReportPrintData` y `CashReportPrintMovement` (según data-model.md §2), un `escapeHtml` interno (`& < > " '`) y `buildCashReportHtml(data): string` cumpliendo I-1…I-12 (estilos en línea propios, `#000`/`#333` sobre `#fff`, sin colores de estado, Arial/Helvetica/"Liberation Sans"). Hacer pasar T008. Comentarios en español de Colombia, enlazando a `A-98` y a research D2/D4
- [ ] T010 [US1] Añadir al mismo `cash-report-print.util.spec.ts` las pruebas de `printCashReportHtml(html, fileTitle)` (jsdom; `window.print` y el iframe simulados con `vi.fn()`/`vi.spyOn`): fija `document.title = fileTitle` y **lo restaura**; agrega un `iframe` oculto (`aria-hidden`) a `document.body` y lo **retira** al `afterprint`, al respaldo de 60 s (`vi.useFakeTimers`) y ante fallo de creación, **una sola vez**; si no hay `contentWindow`/`contentDocument` **no lanza** y deja el DOM idéntico; dos llamadas seguidas no dejan iframes residuales; no toca `body.classList` (contrato §4). Ver que fallan
- [ ] T011 [US1] Implementar `printCashReportHtml` en `cash/services/cash-report-print.util.ts` (mismo archivo que T009) según contrato §4: iframe oculto, `doc.open()/write()/close()`, esperar `complete`/`load`, `win.focus(); win.print()`, limpieza única (retirar iframe **y** restaurar título) en `afterprint` · 60 s · fallo. Comentar la duplicación deliberada del patrón de `receipt.util.ts` (research D3: no se toca la impresión de recibos). Hacer pasar T010

### Store: `imprimirReporte()`

- [ ] T012 [US1] Escribir primero en `cash/services/cash-session.store.spec.ts` las pruebas del nuevo `imprimirReporte()`: arma `CashReportPrintData` desde las señales (negocio, caja, cajero, apertura, cierre, fondo inicial, ventas por método, cambio entregado `null` si es 0, ingresos/egresos/retiros, efectivo esperado/contado, diferencia + etiqueta, observación, movimientos) con **las mismas cifras** que la pantalla (I-12); `fileTitle = <slug(negocio)>-<DD-MM-YYYY>` usando `closed_at` (**no** la fecha de hoy); cae en `cierre-turno` si el slug queda vacío; **no** añade/quita clases en `body`; llama a `buildCashReportHtml` y `printCashReportHtml` (simuladas con `vi.mock`/`vi.spyOn` del módulo); el título de la pestaña queda restaurado. Ver que fallan (hoy usa `body.printing-cash-report` + `window.print()`)
- [ ] T013 [US1] Reescribir `imprimirReporte()` en `cash/services/cash-session.store.ts` (≈ líneas 545-567): armar `CashReportPrintData` desde sus señales (`cajaLabel`, `cajero`, `aperturaFmt`, `cierreFmt`, `indicadores`, `efectivoEsperado`, `diffLabel`, `movimientosView`, `fmt`, `close_note`), calcular `fileTitle` con el mismo `slugify`/`tenantDate`/respaldo actual y llamar a las dos funciones de `cash-report-print.util.ts`; **eliminar** `body.classList` y el listener `afterprint` propio; actualizar el comentario de documentación citando **A-98**. Hacer pasar T012. La firma y la llamada desde `cash-report.component.ts` **no cambian**

### Retiro del mecanismo reemplazado (research D6)

- [ ] T014 [P] [US1] Retirar de `../pos-heladeria/src/app/modules/dashboard/layout/dashboard-layout.component.ts` las reglas `:host-context(body.printing-cash-report)` (spec 087) que ocultaban sidebar/header; antes, `grep -rn "printing-cash-report" ../pos-heladeria/src` para confirmar que no queda otra referencia y que **ninguna prueba** (`dashboard-layout.component.spec.ts` incluida) lo congela; si alguna apareciera, actualizarla **citando A-98** en el mismo cambio. No tocar nada más del layout
- [ ] T015 [P] [US1] Retirar el `@media print` propio de `cash/components/cash-report.component.ts` (queda sin efecto con el iframe); **conservar** los `print:hidden` de los botones (inocuos y correctos ante Ctrl+P). Verificar que `cash-report.component.spec.ts` sigue en verde (solo simula `imprimirReporte` con `vi.fn()`)

### Verificación de la historia

- [ ] T016 [US1] Correr `npx ng test --watch=false` y `npx ng build` en `../pos-heladeria` (verdes, sin errores de tipos) y luego la **Ruta A** del quickstart §3 en Brave (perfil limpio **y** perfil del usuario; este último **requiere al usuario**, y si no se puede hacer se anota en "Limitaciones"): pasos 1–4 (vista previa legible sin menú/encabezado/botones, PDF guardado `<negocio>-<DD-MM-YYYY>` legible en visor externo con texto copiable, turno sin movimientos). Registrar capturas, PDF, navegador/perfil y **el título sugerido por Brave al imprimir desde el iframe** (riesgo D5; si difiere de `fileTitle`, ajustar `printCashReportHtml`/store antes de cerrar) en `implementation-notes.md`

**Checkpoint**: la Historia 1 funciona y es comprobable sola (MVP). La comprobación automática (Fase 5) y las 4 combinaciones (Fase 5) la refuerzan.

---

## Phase 4: User Story 2 — Reimprimir desde el historial de turnos (Priority: P1)

**Goal**: el reporte abierto desde el historial imprime **igual de legible** que al cerrar, con la fecha de **cierre del turno** en el nombre del archivo, y la pantalla vuelve intacta al cancelar (FR-002, FR-007).

**Independent Test**: desde el historial abrir un turno cerrado de otro día, imprimir en Brave y verificar vista previa y PDF; cancelar y pulsar "Volver al historial" (quickstart §3, Ruta B).

- [ ] T017 [US2] Añadir a `cash/services/cash-session.store.spec.ts` las pruebas de la **ruta historial**: con `reportContext` `history` y un `shift` cuyo `closed_at` es de un **día distinto a hoy** (`vi.useFakeTimers`/`vi.setSystemTime`), `imprimirReporte()` produce el mismo `CashReportPrintData` y un `fileTitle` con la **fecha de cierre** (no la de hoy); con `reportContext` `live` y `history` el resultado estructural es equivalente; dos llamadas seguidas (imprimir → cancelar → imprimir) no acumulan iframes ni cambian el título; tras imprimir un reporte recién cerrado y abrir otro desde el historial no queda estado residual (FR-007). Si T012 ya cubre algún caso, no duplicarlo: referenciarlo
- [ ] T018 [US2] Si T017 revela una divergencia entre rutas (`live` vs `history`) en `cash/services/cash-session.store.ts` o en `cash/components/cash-report.component.ts`, corregirla ahí (las dos rutas deben seguir pasando por `imprimirReporte()` sin bifurcar); si no hay divergencia, dejar anotado en `implementation-notes.md` que ambas rutas comparten acción y dato (research, "Hechos verificados")
- [ ] T019 [US2] **Ruta B** del quickstart §3 en Brave (pasos 5–6): Historial → turno de **otro día** → imprimir ⇒ mismo resultado que la Ruta A, PDF con la **fecha de cierre**; **cancelar** el diálogo y pulsar "← Volver al historial" ⇒ pantalla normal (menú, encabezado, colores) sin recargar y sin iframes residuales (DevTools → Elements). Registrar evidencia en `implementation-notes.md`

**Checkpoint**: Historias 1 y 2 funcionan, ambas rutas con el mismo comportamiento.

---

## Phase 5: User Story 3 — Legibilidad garantizada y verificada de verdad (Priority: P2)

**Goal**: la legibilidad no depende del tema, de los fondos ni del tamaño, y queda demostrada con una comprobación **automática repetible** (con control negativo) y con la vista previa real en las 4 combinaciones (FR-004, FR-005, FR-010).

**Independent Test**: `npm run verify:cash-report-print` en verde + pasos 7–9 del quickstart §3 en Brave.

- [ ] T020 [US3] Crear `../pos-heladeria/scripts/verify-cash-report-print.mjs` (NUEVO; la carpeta `scripts/` **no existe**, crearla) según research D7: (1) transpilar `cash-report-print.util.ts` con el `typescript` ya instalado y cargar `buildCashReportHtml`; (2) generar documentos con datos de prueba: **sin movimientos, 6, 70 movimientos y textos con caracteres especiales**; (3) con **Playwright** (`executablePath` desde `BRAVE_BIN`/`CHROME_BIN`, por defecto Brave) y, para **cada** combinación {`prefers-color-scheme` claro, oscuro} × {`printBackground` `true`, `false`}, emular medio `print` y `page.pdf`; (4) verificar con `pdftotext` que el texto esperado está (negocio, cajero, totales, cada fila) y con `pdftoppm` (gris) que **cada hoja** tiene al menos un 0,5 % de píxeles con luminancia < 128 (100 dpi en gris; valor provisional que se calibra en la primera ejecución con el control negativo y queda fijo como constante nombrada en el script), que 70 movimientos dan ≥ 2 hojas todas con contenido, y que las 4 combinaciones producen el mismo texto y una cobertura de tinta que difiere como máximo un 5 % relativo entre sí; (5) **control negativo**: el mismo reporte con texto blanco debe **fallar** la comprobación, y si pasara el script aborta con error; (6) si faltan `pdftotext`/`pdftoppm`/navegador, salir con código ≠ 0 y mensaje claro (no pasar en silencio); imprimir tabla `combinación → hojas / texto OK / tinta OK`; código ≠ 0 nombrando la combinación culpable. Registrar en `implementation-notes.md` los valores medidos (reporte normal y control negativo) que justifican el umbral elegido
- [ ] T021 [US3] Añadir en `../pos-heladeria/package.json` el script `"verify:cash-report-print": "node scripts/verify-cash-report-print.mjs"` (sin dependencias nuevas; Principio IX) y ejecutarlo: debe salir 0 con las 4 combinaciones en verde **y** el control negativo fallando como se espera. Registrar la salida en `implementation-notes.md` (sección "Verificación automática")
- [ ] T022 [US3] **Garantía transversal** del quickstart §3 en Brave (pasos 7–10), con perfil limpio y del usuario: (7) las **4 combinaciones** {tema claro/oscuro del sistema} × {Gráficos de fondo activado/desactivado} dan resultado idéntico; (8) turno de **60+ movimientos** ⇒ varias hojas, ninguna en blanco, filas sin partirse, encabezado de tabla repetido; (9) **imprimir dos veces seguidas** ⇒ ambas legibles, sin iframes residuales; (10) medir SC-006 (< 30 s desde ver el reporte hasta tener el PDF, sin ayuda técnica, en ambas rutas; cronometrado por el usuario, con el quickstart §3 como guion). Los pasos con el **perfil del usuario** **requieren al usuario**; si no se pueden hacer, anotarlo en "Limitaciones". Si una extensión/escudo inevitable altera la salida, documentarlo en "Limitaciones" (Edge Cases de la spec)
- [ ] T023 [P] [US3] **Regresión de otras impresiones** (FR-008, SC-005; quickstart §4): `git diff develop -- src/app/modules/tables/services/receipt.util.ts src/app/modules/tables/pages/table-qr-sheet.component.ts` en `../pos-heladeria` debe estar **vacío**; imprimir en Brave un recibo de venta, un recibo de mesa y la hoja de QR y compararlos con la línea base capturada en T003; registrar resultado en `implementation-notes.md`

**Checkpoint**: las tres historias verificadas; el riesgo de reincidencia queda cubierto por la comprobación automática repetible.

---

## Phase 6: Polish & Cross-Cutting (documentación y trazabilidad, `pos-specs`)

**Propósito**: cerrar la trazabilidad (Principio XII) y la relación con la spec 089 (FR-011).

- [ ] T024 [P] Ampliar **A-98** en `specs/000-reconocimiento/registro-de-anomalias.md` (línea ≈ 3034) con la **causa confirmada** en T007, el cambio aplicado (reporte como documento aislado en iframe; retiro de `printing-cash-report` y del `@media print`) y cómo se verificó; mantener el formato de A-94…A-97
- [ ] T025 [P] Añadir en `specs/089-fix-adicionales-cierre-mesa-caja-pos/spec.md` (Historia 5) y en `specs/089-fix-adicionales-cierre-mesa-caja-pos/tasks.md` (T070 y la fase de US5) una **nota breve de reemplazo** apuntando a la spec 090, **sin reescribir su historial** (research D10, FR-011)
- [ ] T026 Completar `implementation-notes.md`: decisiones tomadas frente a research (D1–D11) y cualquier ajuste (p. ej. título sugerido por Brave, D5), evidencia de T004–T007, T016, T019, T020–T023, la limitación de que **ningún test unitario ejecuta impresión real**, y la nota de que la rama `fix/089-cash-report-print` queda **sin fusionar y reemplazada** (borrarla es decisión del usuario)
- [ ] T027 Validación final: `npx ng test --watch=false` y `npx ng build` en `../pos-heladeria`; `npm run verify:cash-report-print`; recorrer `quickstart.md` §5 (tabla de criterios de salida SC-001…SC-006 y FR-009/FR-011) marcando cada fila con su evidencia; `git status` en `../pos-heladeria` y `pos-specs` para dejar lista la agrupación de commits sugerida (**sin commitear** salvo que el usuario lo pida)

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Fase 1)**: sin dependencias (T002 antes de T003, porque T002 crea `implementation-notes.md`; ambos tras T001).
- **Diagnóstico (Fase 2)**: depende de T001–T003 y **BLOQUEA** todas las historias; orden estricto T004 → T005 → T006 → T007 (E0, luego E1, luego hipótesis, luego decisión).
- **US1 (Fase 3)**: depende de T007. Orden interno: T008 → T009 → T010 → T011 → T012 → T013 → (T014 ∥ T015) → T016. T009 y T011 tocan **el mismo archivo** (no paralelos); T008 y T010 también.
- **US2 (Fase 4)**: depende de T013 (el store ya reescrito). T017 comparte `cash-session.store.spec.ts` con T012: va después de T012 y T013, nunca en paralelo.
- **US3 (Fase 5)**: T020 depende de T009 (el generador existe); T021 de T020; T022 de T016 y T019; T023 es independiente tras T013.
- **Polish (Fase 6)**: T024 necesita T007; T025 es independiente; T026 y T027 al final.

### User Story Dependencies

- **US1 (P1)**: tras el diagnóstico; sin dependencias de otras historias. Es el MVP.
- **US2 (P1)**: reutiliza la implementación de US1 (misma acción); añade pruebas y verificación de la ruta del historial y de FR-007.
- **US3 (P2)**: requiere el generador de US1 (la comprobación automática lo carga); no cambia comportamiento, lo demuestra.

### Parallel Opportunities

- Fase 1: ninguna (T002 → T003 en secuencia).
- Fase 3: T014 ∥ T015 (archivos distintos) una vez hecho T013.
- Fase 4: ninguna; T017 → T018 → T019 en secuencia.
- Fase 5: T023 ∥ T020/T021 (verificación de `git diff`/recibos es independiente del script).
- Fase 6: T024 ∥ T025.

### Parallel Example

```bash
# Tras T013 (store reescrito), en paralelo (archivos distintos):
Task: "T014 Retirar reglas printing-cash-report de dashboard-layout.component.ts"
Task: "T015 Retirar @media print de cash-report.component.ts"

# En Fase 5, en paralelo:
Task: "T020/T021 Script verify-cash-report-print.mjs + package.json"
Task: "T023 Regresión: git diff vacío en receipt.util.ts y table-qr-sheet.component.ts + recibos en Brave"
```

---

## Implementation Strategy

### MVP primero (US1)

1. Fase 1 (rama y prerrequisitos) → Fase 2 (**diagnóstico real en Brave, registrado**).
2. Fase 3 completa: generador puro + impresión en iframe + store + retiro del mecanismo antiguo.
3. **DETENER y VALIDAR** en Brave (T016): si la vista previa y el PDF se ven legibles, el defecto principal está resuelto.

### Entrega incremental

1. + US2: confirmar la ruta del historial (mismo código; pruebas de fecha de cierre y de ausencia de estado residual).
2. + US3: comprobación automática con control negativo y las 4 combinaciones en Brave; regresión de recibos/QR.
3. Polish: A-98 ampliada, nota de reemplazo en 089, notas de implementación y validación final.

### Reglas de cierre

- **Nada se da por resuelto sin la vista previa real de Brave** (FR-010): la comprobación automática la complementa, no la sustituye (es justo el error de la spec 089).
- Si el diagnóstico (T007) contradice el diseño (research D1), se actualiza `research.md` **antes** de seguir.

---

## Notes

- [P] = archivos distintos, sin dependencias pendientes.
- Etiqueta [USn] trazable a `spec.md`; Setup, Diagnóstico y Polish no llevan etiqueta de historia.
- Cada prueba se escribe primero y debe fallar antes de implementar.
- Sin cambios de backend, datos ni contratos HTTP; la reversión es revertir los commits del frontend.
- Commits solo cuando el usuario lo pida (Principio XV).
