# Quickstart — Validación de la spec 090

**Spec**: [spec.md](./spec.md) · **Contrato**: [contracts/cash-report-print.md](./contracts/cash-report-print.md) · **Modelo**: [data-model.md](./data-model.md) · **Decisiones**: [research.md](./research.md)

Guía para comprobar de extremo a extremo que el reporte de caja se imprime legible. Los detalles de implementación viven en `tasks.md`. Regla de la spec: **nada se da por resuelto sin la vista previa real de Brave** (FR-010); la comprobación automática la complementa, no la sustituye.

## 0. Prerrequisitos

- `pos-heladeria` en la rama `fix/090-cash-report-print-isolated` (desde `develop`; Principio XIV; la tarea de creación queda en `tasks.md`). `pos-backend` **sin cambios**.
- Un negocio de pruebas con: un turno de caja **cerrado con movimientos** (ingresos, egresos, retiros y ventas por más de un método de pago), un turno cerrado **sin movimientos**, un turno cerrado de un **día anterior** (para el historial) y uno con **60 o más movimientos** (varias hojas). El de 60 puede armarse con un script de datos de desarrollo; no se toca producción.
- **Brave** de escritorio (`/usr/bin/brave-browser`), y para comparar un **perfil limpio** (`brave-browser --user-data-dir=$(mktemp -d)`, sin extensiones) además del perfil habitual del usuario.
- Poppler para la comprobación automática: `pdftotext` y `pdftoppm` (`sudo apt install poppler-utils` si faltan).

Comandos de verificación automática:

```bash
# pos-heladeria
npx ng test --watch=false                              # suite completa (Vitest), en verde
npx ng build                                           # sin errores de tipos
npm run verify:cash-report-print                       # §2
```

## 1. Diagnóstico en Brave (FR-009) — se hace ANTES de corregir

Se ejecuta sobre **dos estados** del código (research D1): **E0** = `develop` sin arreglo; **E1** = `develop` + `fix/089-cash-report-print`.

1. Levantar la app en cada estado (`ng serve`) y abrir el reporte de un turno cerrado con movimientos.
2. **Perfil limpio**: "Imprimir / Exportar reporte" ⇒ en la **vista previa real**, con "Gráficos de fondo" **desactivado** (valor por defecto): ¿se ve el contenido? Captura.
3. Repetir con **Gráficos de fondo activado**, y de nuevo con el **perfil del usuario** (con sus extensiones y Shields).
4. Repetir con el SO/navegador en **modo oscuro** (o `Emulation.setAutoDarkModeOverride` por CDP).
5. Con DevTools → Rendering → `Emulate CSS media type: print`, anotar el **color calculado** (`color`, `background-color`) del texto y de sus ancestros, y si alguna capa `fixed` cubre la hoja.
6. Comprobar qué versión del bundle corre el navegador del usuario (service worker / hash de `main-*.js`) frente a `develop`.
7. **Registrar en `implementation-notes.md`**: qué hipótesis (H1–H5 de research D1) se confirma, con capturas y colores calculados, y por qué el arreglo de 089 no bastó (o no llegó). **No se escribe código de la corrección hasta tener este registro.**

Salida esperada: una fila de la tabla "Regla de decisión" de research D1, anotada y fechada. Si difiere de las cinco hipótesis, se actualiza `research.md` primero.

## 2. Comprobación automática a PDF sin depender de fondos (FR-010, SC-002, SC-003, SC-004)

`npm run verify:cash-report-print` genera el documento con datos de prueba (sin movimientos, 6 movimientos, 70 movimientos, textos con caracteres especiales) y, **para cada combinación** {tema claro, oscuro} × {gráficos de fondo activados, desactivados}, lo imprime a PDF emulando el medio `print` en el navegador indicado (`BRAVE_BIN`/`CHROME_BIN`; por defecto Brave).

| Comprobación por PDF | Criterio |
|---|---|
| Texto presente | `pdftotext` recupera negocio, cajero, totales y cada fila esperada (SC-002) |
| **Texto visible** | `pdftoppm` (gris): cada hoja tiene ≥ 0,5 % de píxeles con luminancia < 128 (100 dpi, valor provisional calibrado con el control negativo); ninguna hoja en blanco (SC-004). *Leer solo el texto no basta: el texto blanco también se extrae* |
| Hojas | 70 movimientos ⇒ ≥ 2 hojas y todas con contenido; ninguna fila partida |
| Equivalencia | las 4 combinaciones producen el mismo texto y una cobertura de tinta que difiere como máximo un 5 % relativo entre sí (SC-003) |
| **Control negativo** | el mismo reporte con texto blanco **falla** la comprobación; si pasara, el script aborta (no sirve) |

Salida: código 0 y una tabla `combinación → hojas / texto OK / tinta OK`; cualquier fallo sale con código ≠ 0 y la combinación culpable.

## 3. Verificación manual en Brave — ambas rutas (SC-001, SC-006)

Con **perfil limpio** y con **perfil del usuario**:

**Ruta A — al cerrar el turno (Historia 1)**
1. Cerrar un turno con movimientos ⇒ **Imprimir / Exportar reporte**.
2. Vista previa: todos los bloques visibles (datos del turno, resumen, arqueo, movimientos), texto oscuro sobre blanco; sin menú lateral, encabezado de la app ni botones.
3. **Guardar como PDF** ⇒ nombre `<negocio>-<DD-MM-YYYY>` (fecha de cierre); abrir el PDF en un visor externo, sin seleccionar ni cambiar contraste: todo legible; copiar el texto recupera totales y movimientos.
4. Turno **sin movimientos** ⇒ encabezados, totales en cero y "No se registraron movimientos en este turno.".

**Ruta B — desde el historial (Historia 2)**
5. Historial → abrir un turno de **otro día** → imprimir ⇒ mismo resultado; el PDF se llama con la **fecha de cierre del turno**, no la de hoy.
6. **Cancelar** el diálogo y pulsar **← Volver al historial** ⇒ pantalla normal (menú, encabezado, colores), sin recargar.

**Garantía transversal (Historia 3)**
7. Repetir pasos 1–3 en las **4 combinaciones** {tema claro/oscuro del sistema} × {Gráficos de fondo activado/desactivado}: el resultado es idéntico.
8. Turno de **60+ movimientos** ⇒ varias hojas, **ninguna en blanco**, filas sin partirse, encabezado de tabla repetido.
9. **Imprimir dos veces seguidas** (vista previa → cancelar → imprimir) ⇒ ambas legibles y sin iframes residuales (DevTools → Elements).
10. Medir SC-006: desde ver el reporte hasta tener el PDF guardado, < 30 s sin ayuda técnica, en ambas rutas.

Anotar resultado, navegador/perfil y capturas en `implementation-notes.md`.

## 4. Regresión — otras impresiones no cambian (FR-008, SC-005)

11. Imprimir un **recibo de venta** (Ventas), un **recibo de mesa** (Mesas/POS) y la **hoja de QR** (Mesas → QR) en Brave ⇒ idénticos a antes (ancho de rollo, contenido, `@page`). Comparar con una captura previa tomada en `develop` antes del cambio.
12. `git diff develop -- src/app/modules/tables/services/receipt.util.ts src/app/modules/tables/pages/table-qr-sheet.component.ts` ⇒ **vacío**.

## 5. Criterios de salida

| SC | Cómo se comprueba |
|---|---|
| SC-001 | §3 pasos 1–9, en Brave, sin tocar ningún ajuste del navegador |
| SC-002 | §2 (texto presente) y §3 paso 3 (copiar texto del PDF) |
| SC-003 | §2 (equivalencia de las 4 combinaciones) y §3 paso 7 |
| SC-004 | §2 (70 movimientos) y §3 paso 8 |
| SC-005 | §4 pasos 11–12 |
| SC-006 | §3 paso 10 |
| FR-009 | §1 registrado en `implementation-notes.md` **antes** del primer commit de código |
| FR-011 | A-98 ampliada con la causa confirmada; nota en la spec 089 (Historia 5, T070) apuntando a la 090 |
