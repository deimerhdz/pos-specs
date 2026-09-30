# Notas de implementación — Spec 090

**Fecha de inicio**: 2026-09-30 · **Rama (pos-heladeria)**: `fix/090-cash-report-print-isolated` (desde `develop`) · **Spec**: [spec.md](./spec.md) · **Plan**: [plan.md](./plan.md) · **Tareas**: [tasks.md](./tasks.md)

## Diagnóstico

_Fase 2 (T004–T007). **Ejecutado el 2026-09-30 en emulación automatizada (Playwright + Brave/Chrome headless); la vista previa real y el perfil del usuario siguen pendientes** (ver "Pendiente del usuario" al final de esta sección)._

### Método

- App real de `pos-heladeria` (ng serve :4200, API local :8000, tenant `heladeria`) con sesión de desarrollo; turno de prueba **de 70 movimientos** (`[090-test]`, cierre 27/09/2026) abierto desde Historial → "Ver reporte".
- Script fuera del repo (`/tmp/.../scratchpad/diag090.mjs`): reproduce exactamente el mecanismo de `imprimirReporte()` de `develop` (`body.printing-cash-report` + título), emula medio `print` y genera PDF con `page.pdf` para las 4 combinaciones {`prefers-color-scheme` claro/oscuro} × {`printBackground` true/false}. Mide con `pdftotext` (texto presente) y `pdftoppm` -gray 100 dpi (% de píxeles con luminancia < 128 **por hoja**; el texto blanco también se extrae, así que solo la tinta demuestra visibilidad).
- **Limitación**: `page.pdf` no es la vista previa del navegador de escritorio (riesgo H2, exactamente lo que falló en 089). Esto es evidencia fuerte de la causa, no la verificación final (FR-010).

### E0 — `develop` sin arreglo (Brave headless)

| Combinación | Hojas | Tinta por hoja | Texto extraído |
|---|---|---|---|
| claro · fondos **desactivados** | **1** | **0 %** (hoja en blanco) | sí |
| claro · fondos activados | **1** | 4,36 % (velo gris sobre el reporte) | sí |
| oscuro · fondos desactivados | 1 | 0 % | sí |
| oscuro · fondos activados | 1 | 4,36 % | sí |

- **Reproducido el defecto**: con el valor por defecto del navegador (sin gráficos de fondo) la hoja sale **en blanco aunque el texto exista en el PDF**; y se imprime **una sola hoja** de las 5 necesarias para 70 movimientos (solo caben 5 filas).
- Capturas de los PDF: `scratchpad/diag/E0-brave/*.pdf` y `*-img-1.png` (no versionadas).

**Bisección (E0, claro, fondos desactivados; tinta de la primera hoja):**

| Experimento (CSS inyectado en medio print) | Tinta |
|---|---|
| base | 0 % |
| ocultar `[class*="inset-0"]` (telón del menú) | **2,86 %** ← vuelve el contenido |
| `position: static` en todo | 1,69 % |
| ocultar sidebar/header | 0 % |
| texto `#000 !important` | 0 % (el texto **no** es blanco: está **tapado**) |
| sin opacity/filter/mix-blend | 0 % |
| sin overflow/altura fija | 0 % en 5 hojas (paginan pero siguen tapadas) |

Colores calculados del texto en `print` (E0): `color: oklch(0.21 0.034 264.665)` (`text-gray-900`), fondo transparente, ancestros `position: static` — **el texto es oscuro, no blanco**. El elemento culpable, leído del DOM: `div.fixed.inset-0.bg-black/40.z-30.lg:hidden` en `dashboard-layout.component.ts`, presente porque `sidebarOpen()` es `true` en escritorio; al paginar A4 (~816 px, **por debajo del umbral `lg`**) su `lg:hidden` deja de aplicar y se pinta, a página completa, encima del reporte.

### E1 — `develop` + `fix/089-cash-report-print` (rama desechable, ya descartada)

| Navegador | Oscurecimiento forzado (`--force-dark-mode`) | 4 combinaciones tema × fondos | Hojas | Tinta por hoja |
|---|---|---|---|---|
| Brave | no | todas idénticas | 5 | 3,40 · 5,17 · 5,19 · 5,16 · 2,59 % |
| Brave | sí | todas idénticas | 5 | ídem |
| Chrome | no | todas idénticas | 5 | ídem |
| Chrome | sí | todas idénticas | 5 | ídem |

Colores calculados (E1): texto `rgb(0,0,0)` sobre `rgb(255,255,255)` en `print`, ancestros estáticos.

### Hipótesis H1–H5 (research D1)

| # | Resultado | Evidencia |
|---|---|---|
| **H1** — el navegador corre código sin el arreglo | **CONFIRMADA (por código y por E0↔E1)** | E0 reproduce el defecto y E1 no. Además `fix/089-cash-report-print` **no está fusionada a `develop`** (PR #93 fue solo `feat/089-qr-addons-per-line`) **y solo existe en local** (`git ls-remote origin` no tiene la rama), así que **nunca pudo estar desplegada** |
| H2 — vista previa real ≠ `printToPDF` | **NO DESCARTADA** | solo se probó emulación; requiere la vista previa real (pendiente del usuario) |
| H3 — oscurecimiento forzado/extensión | **Descartada en emulación** | E1 idéntico con `--force-dark-mode` en Brave y Chrome; extensiones/perfil del usuario **no probados** |
| H4 — `prefers-color-scheme: dark` | **Descartada** | E0 y E1 idénticos entre claro y oscuro |
| **H5** — capa `fixed` que cubre la hoja | **CONFIRMADA** | bisección: el telón `fixed inset-0 bg-black/40 lg:hidden` |

Nota: la causa confirmada coincide con la que 089 ya había documentado (telón + `h-screen overflow-hidden`); lo que faltó fue **integrarla/desplegarla**, no descubrirla.

### Decisión (T007) — 2026-09-30

Fila de la regla de decisión de research D1 que aplica: **H1 + H5 → "basta integrar 089"**, con dos reservas que la regla no resuelve sola: (a) el usuario reporta haber probado el arreglo y que falla (no descartado que **su** vista previa real o su perfil difieran de la emulación: H2/H3 con su perfil), y (b) el arreglo de 089 depende de `!important` y del shell, frágil ante capas nuevas (motivo del diseño D2).

**Por tanto la Fase 3 se detiene aquí** (regla explícita de T007) a la espera de la decisión del usuario; no se ha escrito código de la corrección.

### Pendiente del usuario (no automatizable)

- Vista previa **real** en Brave de escritorio (perfil del usuario y perfil limpio) con E0 y E1, incl. "Gráficos de fondo" activado/desactivado y su tema del SO.
- Confirmar qué versión del bundle corre su navegador (H1 en su entorno) y si el arreglo de 089 estaba en el código que probó.

## Resultado (2026-09-30)

Tras el diagnóstico el usuario vio el reporte y pidió quitar dos barras de scroll (eje X y derecha de la hoja) y la barra superior de la página. Se resolvió por la ruta "integrar 089" y **no** por el iframe aislado de D2:

- `7b71adb` `fix(layout): release the shell height, overflow and mobile backdrop when printing the cash report` — equivale al commit `289e6ea` de `fix/089-cash-report-print` (aplicado con `cherry-pick -n`), con su prueba.
- `d17ed49` `fix(cash-register): hide the cash page top bar when printing the report` — solo la línea `print:hidden` de `cash-page.component.ts` del commit `e6a6dc3`.

Ambos en `pos-heladeria`, rama `fix/090-cash-report-print-isolated` (el nombre quedó heredado del plan; **no** contiene el iframe). El usuario confirmó que con esto queda arreglado.

Verificación hecha: PDF con Brave headless, 4 combinaciones tema × fondos, 70 movimientos → 5 hojas con tinta, sin barras de scroll ni barra superior; `dashboard-layout.component.spec.ts` 12/12.

**No ejecutado** (la Fase 3 en adelante de `tasks.md` queda sin hacer; T003–T007 solo parcialmente): documento aislado en iframe (T008–T016), pruebas de historial (T017–T019), script `verify-cash-report-print` (T020–T022), regresión de recibos/QR (T023), A-98 ampliada y nota en 089 (T024–T026). Sigue sin verificarse en la vista previa real de Brave con el perfil del usuario (FR-010) y `fix/089-cash-report-print` sigue sin fusionarse (el layout y la barra se aplicaron por separado). Si el defecto reaparece, el diseño D2 sigue vigente en `research.md`.

## Decisiones

_(Se completa en T026.)_

## Verificación automática

_(Se completa en T021.)_

## Verificación manual en Brave

### Línea base FR-008

_(T003: recibo de venta, recibo de mesa y hoja de QR en `develop`, antes de cambiar código. Pendiente.)_

## Limitaciones

- Ninguna prueba unitaria ejecuta impresión real; la impresión solo se da por verificada con la vista previa y el PDF guardado en Brave (FR-010).

---

## Registro de ejecución

### T001 — Rama (2026-09-30)

- `../pos-heladeria` estaba en `develop` con árbol limpio. Se creó `fix/090-cash-report-print-isolated` desde `develop`.
- `git merge-base --is-ancestor fix/089-cash-report-print HEAD` → código **1** (la rama de 089 **no** está contenida), como exige la tarea.

### T002 — Prerrequisitos (2026-09-30)

| Prerrequisito | Resultado |
|---|---|
| Brave | `/usr/bin/brave-browser` |
| Chrome | `/usr/bin/google-chrome` |
| `pdftotext` / `pdftoppm` | `/usr/bin/pdftotext`, `/usr/bin/pdftoppm` |
| Playwright | `npx playwright --version` → 1.60.0 (desde `../pos-heladeria`) |
| `scripts/` en `pos-heladeria` | no existe (se crea en T020) |
| Stack de desarrollo | `ng serve` en :4200, API local (`fastapi`) en :8000, Postgres en :5432 (contenedor `pos`), Redis. El contenedor `pos-api` está en bucle de reinicio; lo que atiende :8000 es un proceso `python3`/`fastapi` local |

**Línea base de `npx ng test --watch=false` en `develop` (antes de cambiar nada): NO está en verde.**
`Test Files 6 failed | 90 passed (96)` · `Tests 19 failed | 1094 passed | 1 skipped (1114)`.

Las fallas son ajenas a caja y a impresión; se registran para no confundirlas con regresiones de esta spec:

| Archivo | Fallas |
|---|---|
| `src/app/app.spec.ts` | 2 |
| `src/app/core/services/auth.service.spec.ts` | 10 (`PushRegistrationService` no inyectable en el TestBed) |
| `src/app/core/services/menu.service.spec.ts` | 1 |
| `src/app/modules/super-admin/services/tenant.service.spec.ts` | 3 (+3 en `verify`) |
| `src/app/modules/tables/components/pos-checkout-panel.component.spec.ts` | 1 |
| `src/app/modules/tables/components/pos-order-panel.component.spec.ts` | 2 |

Criterio para T016/T027: el resultado final debe tener **exactamente esas mismas 19 fallas y ninguna más**, más las pruebas nuevas en verde.

### T003 — Datos de prueba (2026-09-30) — parcial

La base de desarrollo (`pos_db`, esquema `heladeria`) tenía **4 turnos cerrados, los 4 sin movimientos**. Se sembraron con SQL (todos marcados `[090-test]` en `close_note`, para limpiarlos con `DELETE FROM heladeria.cash_shifts WHERE close_note LIKE '[090-test]%'`; los movimientos caen por `ON DELETE CASCADE`):

| Turno (id) | Contenido |
|---|---|
| `…000000090001` | cierre 28/09/2026 (día anterior), 6 movimientos, con `<img onerror>`, `&`, comillas y `<b>` en notas y observación |
| `…000000090002` | cierre 27/09/2026, **70 movimientos** (notas largas) |
| turnos previos | 4 sin movimientos (cubren el caso "sin movimientos") |

Limitación: no se sembraron **ventas** por varios métodos de pago (el turno usa solo ingresos/egresos/retiros). **Pendiente**: línea base FR-008 (recibo de venta, recibo de mesa, hoja de QR en `develop`), porque requiere la vista previa real.

## Entorno de verificación (hallazgos)

- El navegador del MCP de chrome-devtools es un **Chrome headless propio** (UA `HeadlessChrome/151`), no el Brave del usuario; no expone puerto CDP utilizable. La automatización se hizo con Playwright + `/usr/bin/brave-browser` y `/usr/bin/google-chrome` headless, reutilizando la sesión de desarrollo de ese navegador (tokens copiados a un archivo privado del scratchpad, no versionado).
- Había 6 pestañas obsoletas de la app (`heladeria.localhost:4200`) en ese Chrome que agotaban el límite de 6 conexiones por host hacia `localhost:8000` y dejaban la caja en "Cargando caja…"; se cerraron con autorización del usuario.

## Bloqueos

- **Fase 3 detenida por regla de T007**: el diagnóstico cae en "basta integrar 089" (H1+H5). Requiere decisión del usuario (ver Diagnóstico → Decisión).
