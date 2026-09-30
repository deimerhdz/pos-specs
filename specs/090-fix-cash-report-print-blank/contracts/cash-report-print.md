# Contrato — Documento imprimible del reporte de caja

**Historias**: 1, 2, 3 · **FR**: 001–008 · **Decisiones**: research D2–D5, D9 · **Anomalía**: A-98

Es el contrato de una **interfaz interna de presentación** (no hay API HTTP nueva). Define qué recibe, qué produce y qué invariantes cumple el documento que se imprime, para que la comprobación automática y la manual tengan un criterio único.

## 1. Interfaz (módulo de TypeScript puro, en `cash-register/services/`)

| Símbolo | Firma | Responsabilidad |
|---|---|---|
| `buildCashReportHtml` | `(data: CashReportPrintData) => string` | Devuelve un documento HTML completo (`<!doctype html>…`). **Puro**: sin DOM, sin Angular, sin red; mismo dato ⇒ mismo HTML |
| `printCashReportHtml` | `(html: string, fileTitle: string) => void` | Crea el iframe oculto, escribe el documento, espera `complete`, imprime, limpia (§4) |
| `CashReportPrintData` | ver [data-model.md](../data-model.md) | Datos ya resueltos a texto |

`CashSessionStore.imprimirReporte()` (mismo nombre y misma llamada desde `cash-report.component.ts`; **no cambia su contrato hacia la pantalla**) arma `CashReportPrintData` desde sus señales, calcula `fileTitle = <slug(negocio)>-<DD-MM-YYYY>` (respaldo `cierre-turno`; fecha de **cierre**), y llama a las dos funciones anteriores.

## 2. Invariantes del documento (lo que la comprobación verifica)

| # | Invariante | FR / SC |
|---|---|---|
| I-1 | **Todo texto es `#000` (o `#333` para rótulos secundarios) sobre fondo `#fff`**; ningún texto blanco, transparente ni gris claro | FR-003 |
| I-2 | Declara `color-scheme: only light` (CSS y `<meta>`), `print-color-adjust: exact` y `forced-color-adjust: none` | FR-004 |
| I-3 | **Ningún texto depende de un fondo**: sin fondos tintados; bordes de línea de 1 px. Con "Gráficos de fondo" activado o no, el resultado es el mismo | FR-004, SC-003 |
| I-4 | Sin colores de estado; la diferencia del arqueo se lee por **texto** ("Cuadre perfecto"/"Sobrante"/"Faltante") y signo | FR-003 |
| I-5 | Contiene, en este orden: encabezado (negocio, "Reporte de cierre — caja", cajero, apertura, cierre), "Resumen financiero", "Arqueo de caja" (+ observación si hay), "Movimientos del turno (N)" | FR-001 |
| I-6 | Turno sin movimientos ⇒ encabezados y totales presentes (en cero) y la línea "No se registraron movimientos en este turno." — nunca un documento vacío | Historia 1, esc. 3 |
| I-7 | `thead` se repite en cada hoja (`display: table-header-group`); ninguna fila se parte (`tr { break-inside: avoid }`); sin alturas fijas ni `overflow` | FR-005, SC-004 |
| I-8 | `@page { size: A4; margin: 10mm }` | FR-005 |
| I-9 | **Sin menú, encabezado de la app, botones ni recursos externos** (ni CSS, fuentes, imágenes o scripts de la app); sin atributos `class` de Tailwind | FR-006 |
| I-10 | Todo texto proveniente de usuario/base está **escapado**; el documento no contiene `<script>` ni manejadores `on*` | research D9 |
| I-11 | `<title>` = `fileTitle` (`<negocio>-<DD-MM-YYYY>`) | FR-006, spec 087 |
| I-12 | Las cifras son las del store, sin recalcular (pantalla = papel) | FR-001 |

## 3. Estructura de contenido (orientativa, no implementación)

```text
<negocio>                                  ← título del documento (h1)
Reporte de cierre — <caja>
Cajero · Apertura · Cierre
──────────────────────────────
Resumen financiero      (tabla 2 columnas: fondo inicial, ventas por método,
                         [cambio entregado], ingresos, egresos, retiros,
                         efectivo esperado)
Arqueo de caja          (efectivo contado · diferencia + etiqueta)
[Observación: …]
Movimientos del turno (N)
  Hora | Tipo | Valor | Usuario | Observación     ← thead repetido por hoja
  …filas…                                          ← sin partirse
```

## 4. Ciclo de `printCashReportHtml` (comportamiento observable)

1. Fija `document.title = fileTitle` (guarda el original).
2. Crea un `iframe` oculto, `aria-hidden`, sin tamaño visible, y lo agrega a `document.body`.
3. Escribe el documento con `doc.open()/write()/close()` y espera `readyState === 'complete'` (o `load`).
4. `win.focus(); win.print()`.
5. **Limpieza** exactamente una vez, en el primero de: `afterprint` del iframe · 60 s (respaldo, igual que el recibo) · fallo al crear el documento. Limpieza = retirar el iframe **y** restaurar `document.title`.
6. Si el iframe no se puede crear (sin `contentWindow`/`contentDocument`): retira el iframe, restaura el título y **no lanza** (la pantalla sigue usable).

**Garantías para la pantalla** (FR-007): no se agregan ni se quitan clases de `body`; no hay iframes residuales; el título original se restaura; imprimir dos veces seguidas funciona (cada llamada crea y retira su propio iframe).

## 5. Lo que este contrato NO cambia

- `receipt.util.ts` y los recibos de venta/mesa; `table-qr-sheet.component.ts` y su `window.print()` (FR-008, SC-005).
- El contenido y las cifras del reporte en pantalla; el nombre de archivo (`<negocio>-<DD-MM-YYYY>`); la acción visible "Imprimir / Exportar reporte".
- Ninguna API, esquema ni dato.
