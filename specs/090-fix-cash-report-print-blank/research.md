# Research — Spec 090

**Fecha**: 2026-09-30 · **Spec**: [spec.md](./spec.md) · **Plan**: [plan.md](./plan.md)

Cada decisión sale de leer el código de `pos-heladeria` (rama `develop`, árbol limpio), la rama `fix/089-cash-report-print`, las specs 087 y 089 y el equipo disponible (Brave, Chrome, Playwright, poppler). No quedó ningún `NEEDS CLARIFICATION`. Lo que **no se puede saber leyendo código** —la causa real en Brave— se deja como la primera tarea del plan con un método y una regla de decisión (D1, FR-009), no como suposición.

## Hechos verificados (punto de partida)

| Hecho | Evidencia |
|---|---|
| La impresión actual es `window.print()` sobre **la misma página de la app**: `CashSessionStore.imprimirReporte()` pone `document.title`, añade `body.printing-cash-report` y llama a `window.print()`; el reporte comparte documento con el shell (`dashboard-layout`: `h-screen overflow-hidden`, `sidebar` `fixed`, telón `fixed inset-0`, `lg:ml-64`). | `cash-session.store.ts:552-567`, `dashboard-layout.component.ts:18-50`, `cash-page.component.ts` |
| Las dos rutas (cierre e historial) son **la misma pantalla** `app-cash-report` y **la misma acción** `store.imprimirReporte()`. | `cash-page.component.ts` (`@case ('report')`), `cash-report.component.ts:94-106` |
| **El arreglo de la spec 089 no está en `develop`.** `fix/089-cash-report-print` (2 commits: `289e6ea` layout, `e6a6dc3` reporte) parte de `231da1e` y **no se fusionó**; el PR #93 (`feat/089-qr-addons-per-line`) solo trajo adicionales, cierre de mesa y total del POS. | `git log develop..fix/089-cash-report-print`, `git merge-base --is-ancestor` |
| El arreglo de 089 pelea contra el shell con CSS `!important` (`:host, :host * { color:#000 !important; background:#fff !important }`), liberando alturas y overflow de tres contenedores y ocultando el telón con `print:hidden`. Se verificó **solo** con Chrome headless y CDP (`printToPDF`, `printBackground:false`); T070 (vista previa real, Firefox) quedó abierta. | `implementation-notes.md` de 089, Fase 7; `git show 289e6ea e6a6dc3` |
| La app **no tiene tema oscuro propio**: no hay `dark:`, `color-scheme` ni `prefers-color-scheme` en `src/`. El "tema oscuro" del usuario es del navegador o del sistema. | búsqueda en `src/` (solo `chart-theme.ts`, ajeno al reporte) |
| El texto del reporte es **oscuro por clase** (`text-gray-900`, `text-gray-500`…) sobre el fondo `bg-gray-50` del shell; no fija colores de impresión en `develop`. | `cash-report.component.ts` |
| Hay un **precedente propio y vigente** de imprimir sin depender del shell: los recibos se generan como documento HTML independiente (`buildReceiptHtml`) y se imprimen en un **iframe oculto** (`printReceiptHtml`), sin Tailwind ni estilos de la app. El usuario no ha reportado fallos en ellos. | `receipt.util.ts:1-10, 357-396` |
| Hay un PWA con service worker (`provideServiceWorker('push-sw.js', …)`): un navegador puede servir un bundle anterior tras un despliegue. | `app.config.ts:44` |
| El equipo tiene Brave (`/usr/bin/brave-browser`), Chrome, Playwright (`devDependency` ya declarada, sin uso actual en el repo), `pdftotext` y `pdftoppm`. | `which`, `package.json` |
| Los datos del reporte que faltan en pantalla ya están resueltos en el store (`cajaLabel`, `cajero`, `aperturaFmt`, `cierreFmt`, `indicadores`, `movimientosView`, `diffLabel`, `fmt`); no hay cambio de backend ni de datos. | `cash-session.store.ts:143-240, 597-625` |

## D1 — Diagnóstico primero, en Brave, con dos estados del código (FR-009)

**Decisión**: la **primera tarea** reproduce el defecto en Brave con la vista previa real y lo registra en `implementation-notes.md` **antes de tocar código**. Se prueban **dos estados del código** porque el resultado decide el alcance:

| Estado | Qué es | Por qué |
|---|---|---|
| **E0** | `develop` actual (sin ningún arreglo de impresión) | La spec 090 asume que el usuario vio el fallo *con* el arreglo de 089, pero ese arreglo **no está en `develop`**; si E0 reproduce y E1 no, la "causa" es que nunca llegó |
| **E1** | `develop` + `fix/089-cash-report-print` (lo que el usuario dice haber probado) | Es el caso que el usuario reportó; si falla aquí, el arreglo de 089 no basta y hay otra causa |

Hipótesis a descartar con evidencia (no se presupone ninguna):

| # | Hipótesis | Cómo se comprueba |
|---|---|---|
| H1 | El navegador del usuario corre **código viejo** (sin el arreglo): el arreglo no se desplegó o el service worker sirve un bundle anterior | Comparar E0 y E1 en vista previa real; revisar en el navegador del usuario qué versión del bundle carga (`Application → Service Workers`, hash de `main-*.js`) |
| H2 | El arreglo sí está, pero la **vista previa real de Brave difiere** de `printToPDF` emulado (p. ej. un elemento `fixed`, un toast o el telón sigue pintándose encima, o `afterprint`/clase de `body` llega tarde) | E1 en vista previa real: capturar el árbol de capas (DevTools → Rendering → `Emulate CSS media type: print`) y el PDF guardado |
| H3 | **Oscurecimiento forzado del navegador o de una extensión** (modo oscuro automático de Chromium/Brave, Dark Reader, Shields) que pone el texto en blanco y, al no imprimirse fondos, lo deja blanco sobre blanco | Mismo flujo con **perfil limpio** (`--user-data-dir` temporal, sin extensiones) frente al **perfil del usuario**; en CDP, `Emulation.setAutoDarkModeOverride` |
| H4 | Un color heredado del tema del sistema: `prefers-color-scheme: dark` del SO cambia colores por defecto del documento | `Emulation.setEmulatedMedia` con `prefers-color-scheme: dark` y color **calculado** de `app-cash-report` (texto y fondo) |
| H5 | Alguna capa `fixed` de la app (toast, `confirm-dialog`, telón) se pinta a página completa en cada hoja | Bisección: ocultar capas una a una en emulación `print` y repetir el PDF |

Para cada hipótesis se guarda: captura de la vista previa, PDF guardado, colores calculados (`color`, `background-color`, `print-color-adjust`) del texto y de sus ancestros, y el perfil usado (limpio / del usuario).

**Regla de decisión** (qué hace el plan según el resultado):

| Resultado del diagnóstico | Acción |
|---|---|
| H1 (E0 falla, E1 funciona en vista previa real) | El defecto era que el arreglo de 089 no llegó. Aun así **se ejecuta D2** si E1 depende de fondos o de tema (SC-003/FR-004 exigen independencia); si E1 pasa las 4 combinaciones, basta integrar 089 y se documenta (spec, Suposiciones) |
| H2 o H5 | Hay una capa del shell que el CSS de 089 no neutraliza: D2 la elimina de raíz (el documento impreso no contiene shell) |
| H3 o H4 | La causa está en el navegador/extensión: D2 + **defensas propias del documento** (D4) y, si una extensión sigue alterando la salida, **limitación conocida documentada** (spec, Edge Cases) |
| Causa distinta a todas | Se actualiza este `research.md` antes de continuar (mismo mecanismo que D13 en 089) |

**Por qué no se asume ya**: en 089 la causa que se "confirmó" difería de las tres hipótesis del plan, y aun así el defecto persiste; repetir el error de declarar resuelto sin vista previa real es justo lo que esta spec existe para evitar.

## D2 — Imprimir el reporte como documento HTML independiente en un iframe oculto

**Decisión**: el reporte se imprime generando un **documento HTML autocontenido** (estilos en línea propios, sin Tailwind, sin hojas de la app, sin shell) y abriéndolo en un **iframe oculto** desde el que se llama a `print()`, el mismo patrón que ya usan los recibos. `CashSessionStore.imprimirReporte()` deja de tocar `body` y de depender de `window.print()` sobre la página.

**Por qué**:
- La falla persistió tras **dos** correcciones que combatían el shell por capas de CSS (ocultar, liberar alturas, forzar `!important`). El shell es la superficie de riesgo: sidebar `fixed`, telón `fixed`, `h-screen overflow-hidden`, márgenes `lg:ml-64`, toasts, diálogos. Un documento aparte **no contiene nada de eso**: no hay qué ocultar, liberar ni neutralizar.
- Cumple por construcción FR-003/FR-004/FR-005/FR-006/FR-007: colores fijados en el propio documento, independiente del tema de la app, de los fondos y del tamaño del reporte; menú, encabezado y botones **no existen** en el documento; al cancelar no queda estado en la pantalla (el iframe se retira).
- Es el patrón con que ya imprimen los recibos de venta y de mesa (estables en producción), no una técnica nueva.
- El contenido del reporte sigue saliendo de los mismos datos (`CashSessionStore`), así que pantalla y papel no divergen en cifras.

**Alternativas consideradas**:
1. **Integrar `fix/089-cash-report-print` tal cual** — descartada como solución única: es lo que el usuario dice haber probado y el defecto persiste; depende de `!important` y de que ninguna capa nueva del shell se pinte encima; nació de una verificación parcial. Queda como **ruta mínima** solo si D1 prueba que la causa era que no se desplegó (regla de decisión).
2. **`window.open` con una ventana nueva** — descartada: bloqueadores de ventanas emergentes (Brave Shields los bloquea con frecuencia), un paso extra para el cajero y el riesgo de que el navegador cierre la ventana antes de imprimir.
3. **Generar un PDF en servidor** — descartada: no hay infraestructura de PDF, añadiría dependencia y backend a un defecto de presentación (Principios V, VI, IX), y la spec lo declara fuera de alcance (sin cambios de backend).
4. **Generar el PDF en el cliente (jsPDF/pdfmake)** — descartada: dependencia nueva (Principio IX) sin necesidad; el navegador ya hace "Guardar como PDF" y el usuario lo usa así.
5. **CSS `@media print` que oculta todo salvo el reporte (`body * { visibility:hidden }`)** — descartada: los elementos `fixed` y los contenedores con `overflow` siguen afectando la paginación; es otra variante de pelear contra el shell.

## D3 — Un solo punto de impresión, sin reutilizar `printReceiptHtml` ni tocarlo

**Decisión**: una función propia del módulo de caja, junto al generador del documento, que crea el iframe, escribe el documento, espera a `complete`, llama a `win.print()` y **retira el iframe** al `afterprint` (con respaldo por tiempo, como hace el recibo). No se modifica `receipt.util.ts` ni se extrae una utilidad compartida.

**Por qué no reutilizar `printReceiptHtml`**: (a) el reporte necesita controlar el **título** para el nombre del archivo (D5) y restaurar el de la pestaña al terminar; `printReceiptHtml(html)` no lo permite; (b) modificarlo o extraerlo **toca la impresión de recibos**, que FR-008/SC-005 exigen sin cambios (Principio V: no hay refactor oportunista). El costo es ~30 líneas de iframe duplicadas, deliberadas y documentadas en el código.

## D4 — Estilo del documento: negro sobre blanco, sin depender de nada externo

**Decisión** (contrato en [contracts/cash-report-print.md](./contracts/cash-report-print.md)):
- `html, body { background:#fff; color:#000 }` y **todo texto hereda `#000`** (sin grises claros: los rótulos secundarios usan `#333`/`#444`, nunca menos contraste que 7:1 sobre blanco).
- `color-scheme: only light` en `:root` y `<meta name="color-scheme" content="only light">`: indica al navegador que el documento **no admite** oscurecimiento automático (defensa frente a H3/H4).
- `-webkit-print-color-adjust: exact; print-color-adjust: exact` y `forced-color-adjust: none`: los colores declarados se respetan con o sin "Gráficos de fondo" (FR-004). Como **ningún texto depende de un fondo** (el fondo es blanco y el texto negro, y los bordes son líneas de 1 px `#000`/`#999`), desactivar fondos no puede ocultar nada.
- **Sin colores de estado** (verde/rojo/índigo) ni fondos tintados: en papel la diferencia del arqueo se comunica con su **texto** ("Cuadre perfecto", "Sobrante", "Faltante") y el signo, no con color (accesible en impresión en blanco y negro).
- `@page { size: A4; margin: 10mm }`. Tablas con `thead { display: table-header-group }` (se repite en cada hoja) y `tr { break-inside: avoid }` (filas sin partirse, FR-005). Sin alturas fijas ni `overflow` (nada recorta al paginar).
- Sin recursos externos (ni fuentes web, ni imágenes, ni íconos): el documento es imprimible sin red y sin esperar cargas. Tipografía: `Arial, Helvetica, "Liberation Sans", sans-serif`.

**Por qué**: es exactamente la lista de causas que D1 puede encontrar (tema, fondos, capas) ya **anulada de antemano** en el propio documento; no hace falta saber cuál era para que ninguna aplique.

## D5 — Nombre del archivo (spec 087, FR-006) y su riesgo

**Decisión**: se conserva el nombre `<negocio>-<DD-MM-YYYY>` con la **fecha de cierre del turno** y el mismo `slugify` y respaldo `cierre-turno` de hoy. El documento del iframe lleva ese valor en `<title>` **y** `document.title` de la pestaña se fija igual mientras dura la impresión y se restaura al `afterprint`/retiro del iframe.

**Riesgo declarado**: no está garantizado qué título usa Chromium/Brave como nombre sugerido al imprimir desde un iframe (el del documento impreso o el de la ventana superior). Por eso se fijan **ambos** (barato y seguro) y el nombre se **comprueba en la vista previa real de Brave** (quickstart §3; FR-010) en ambas rutas y con un turno de un día distinto al de hoy. Si el resultado difiriera, se ajusta aquí antes de dar por terminada la tarea.

## D6 — Qué se hace con el código de 087/089 que queda sin uso

**Decisión**: al pasar a iframe, el mecanismo de spec 087 (`body.printing-cash-report` + las reglas `:host-context(...)` de `dashboard-layout.component.ts`) y el `@media print` propio de `cash-report.component.ts` **dejan de tener efecto**. Se **retiran** en el mismo cambio, citando **A-98**, porque son la mitad del mismo mecanismo reemplazado (FR-011), no una refactorización ajena (Principio V). Los `print:hidden` de los botones de la pantalla se conservan (inocuos y todavía correctos si alguien usa Ctrl+P). **No se integra** ninguna parte de `fix/089-cash-report-print` (layout, cash-page, `!important`): la rama se deja sin fusionar y se documenta como reemplazada (FR-011); borrarla es decisión del usuario.

**Condición**: si D1 cae en la fila "basta integrar 089", esta decisión y D2–D4 se revisan antes de ejecutar (la regla de decisión de D1 manda).

## D7 — Comprobación automática repetible que imprime a PDF sin depender de fondos (FR-010)

**Decisión**: un script Node en `pos-heladeria/scripts/verify-cash-report-print.mjs` (+ script npm `verify:cash-report-print`) que:

1. Transpila el generador del documento con el `typescript` ya instalado (el módulo es **TS puro, sin imports de Angular**, para poder cargarse así) y lo invoca con **datos de prueba** (turno sin movimientos, con 6, con 70 movimientos, con texto con caracteres especiales).
2. Abre cada documento en el navegador indicado (`CHROME_BIN`/`BRAVE_BIN`; por defecto Brave) con **Playwright** (ya declarado como `devDependency`; sin dependencia nueva) y, para **cada combinación** {`prefers-color-scheme` claro, oscuro} × {`printBackground` `true`, `false`}, emula el medio `print` y genera el PDF (`page.pdf`).
3. Verifica con `pdftotext` que **el texto esperado** (negocio, cajero, totales, cada fila) está presente; y con `pdftoppm` (rasterizado en gris) que **cada hoja** tiene al menos un 0,5 % de píxeles con luminancia < 128 (100 dpi; umbral provisional que se calibra con el control negativo y queda como constante nombrada) y que **ninguna** sale en blanco. El segundo control es imprescindible: **el texto blanco sobre blanco también se extrae con `pdftotext`**, así que solo leer texto no detectaría este defecto.
4. Incluye un **control negativo**: el mismo reporte con texto blanco debe **fallar** la comprobación; si pasara, el script es inútil y se aborta.

**Por qué no una prueba unitaria**: ningún test unitario (Vitest + jsdom) ejecuta impresión ni genera PDF. **Por qué no contra la app completa**: exigiría backend, sesión y datos; el documento aislado ya es **el** artefacto que se imprime, y la comprobación manual en Brave (quickstart §3) cubre el flujo real de extremo a extremo.

**Herramientas del sistema** (no dependencias del proyecto): `pdftotext` y `pdftoppm` (poppler). Se documentan como prerrequisito en el quickstart; si faltan, el script lo dice y sale con error, no pasa en silencio.

## D8 — Pruebas de unidad

**Decisión** (`ng test`: `@angular/build:unit-test` con Vitest y jsdom, como el resto del repo; `vi.fn()` en lugar de Jasmine):
- El generador: todas las secciones presentes (datos del turno, resumen, arqueo, observación, movimientos); turno sin movimientos ⇒ mensaje y totales en cero; **escape de HTML** en los textos libres (notas, observación, nombre de cajero, negocio); sin `class="…"` de Tailwind ni recursos externos; contiene `color-scheme`, `print-color-adjust`, `@page`, `thead`/`break-inside`; `<title>` = nombre de archivo.
- `imprimirReporte()`: invoca el punto de impresión con el título `<negocio>-<DD-MM-YYYY>` usando `closed_at` (no la fecha de hoy), cae en `cierre-turno` si el slug queda vacío, **no** añade clase a `body` y restaura `document.title`; vale igual con `reportContext` `live` e `history`.
- Verificado en `develop`: **ninguna** prueba congela `printing-cash-report` ni `imprimirReporte()` (solo `cash-report.component.spec.ts` lo simula con `vi.fn()`), así que el retiro de D6 no rompe pruebas previas; si aparece alguna al implementar, se actualiza citando A-98.
- Ninguna prueba ejecuta impresión real (se declara en `implementation-notes.md`, como en 089).

## D9 — Seguridad del documento generado

**Decisión**: el documento se construye con un **escapador de HTML** aplicado a **todo** texto que venga del usuario o de la base (nombre del negocio, cajero, categoría y descripción de movimientos, observación del cierre, nombre de la caja, nombres de métodos de pago). Se escribe con `doc.write` en un iframe del **mismo origen** y sin scripts en el documento. **Por qué**: hoy Angular escapa por defecto; al armar HTML a mano esa protección desaparece, y una nota de movimiento con `<script>` o `<img onerror>` se ejecutaría en el origen de la app. Hay prueba unitaria explícita (D8).

## D10 — Ramas y verificación de relación con 089 (Principio XIV, FR-011)

**Decisión**: en `pos-heladeria`, una rama `fix/090-cash-report-print-isolated` **desde `develop`** (la rama actual; no desde `fix/089-cash-report-print`, que no se hereda). `pos-backend` **no se toca** (sin rama). `pos-specs` sigue en su flujo habitual. FR-011: A-98 se amplía con la causa confirmada (D1) y una nota en la spec 089 (Historia 5 / T070) marca esa parte como **reemplazada por la 090**, sin reescribir su historial.

## D11 — Fuera de alcance confirmado

Recibo de venta, recibo de mesa y hoja de QR (incluido `table-qr-sheet.component.ts`, que sigue con `window.print()` y su propio `@media print`), Firefox/Safari/móviles/impresoras térmicas, cualquier cambio de backend o de datos, el contenido del reporte (cifras y secciones **no cambian**; solo su presentación impresa), y el nombre de archivo (se conserva).
