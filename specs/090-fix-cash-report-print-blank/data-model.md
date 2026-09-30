# Modelo de datos — Spec 090

**Fecha**: 2026-09-30 · **Spec**: [spec.md](./spec.md) · **Decisiones**: [research.md](./research.md) D2, D4, D8, D9

## 1. Cambios de esquema y de datos persistentes

**Ninguno.** No hay migración, columna, tabla, índice, valor por defecto ni cambio de API. Ningún dato histórico se lee distinto ni se reescribe (Principio VII: turnos, movimientos y arqueos cerrados quedan idénticos; solo cambia cómo se dibuja su impresión). No aplica estrategia de migración ni de reversión de datos (Principio VIII); la reversión del cambio es **de código** (revertir los commits del frontend).

## 2. Modelo de presentación (en memoria, cliente web)

Es la única estructura nueva: el **contenido del reporte ya resuelto a texto**, que el store entrega al generador del documento. Vive solo en memoria durante la impresión, **no se persiste ni viaja a ninguna API**. Se define en un módulo de TypeScript **sin importaciones de Angular** (para que el script de comprobación pueda cargarlo; research D7).

### `CashReportPrintData`

| Campo | Tipo | Origen (hoy, en `CashSessionStore`) | Nota |
|---|---|---|---|
| `negocio` | `string` | `tenantInfo.businessName()` | Nombre visible; el slug del archivo se calcula aparte |
| `caja` | `string` | `cajaLabel()` | |
| `cajero` | `string` | `cajero()` | |
| `apertura` | `string` | `aperturaFmt()` | Ya formateado con la zona horaria del negocio |
| `cierre` | `string` | `cierreFmt()` | Ídem |
| `fondoInicial` | `string` | `fmt(fondoInicial)` | Moneda ya formateada (`$ 1.234`) |
| `ventas` | `{ nombre: string; total: string }[]` | `indicadores().ventas` | Una fila por método de pago, en el orden actual |
| `cambioEntregado` | `string \| null` | `indicadores().cambioEntregado` | `null` si es 0 (hoy la fila solo aparece si es > 0) |
| `ingresos`, `egresos`, `retiros` | `string` | `indicadores()` | |
| `efectivoEsperado` | `string` | `efectivoEsperado()` | |
| `efectivoContado` | `string` | `fmt(contado)` | |
| `diferencia` | `{ valor: string; etiqueta: string }` | `fmt(diferencia)` + `diffLabel()` | `etiqueta` ∈ {"Cuadre perfecto","Sobrante","Faltante"}; **sin color** |
| `observacion` | `string` | `close_note` | Vacío ⇒ no se imprime la línea |
| `movimientos` | `CashReportPrintMovement[]` | `movimientosView()` | Vacío ⇒ mensaje "No se registraron movimientos en este turno." |
| `fechaArchivo` | `string` | `tenantDate.transform(closed_at, 'dd-MM-yyyy')` | Fecha de **cierre**, nunca la de hoy |

### `CashReportPrintMovement`

| Campo | Tipo | Origen |
|---|---|---|
| `hora` | `string` | `MovementRow.hora` |
| `tipo` | `string` | `MovementRow.label` |
| `valor` | `string` | `MovementRow.montoFmt` (con su signo `+`/`-`) |
| `usuario` | `string` | `MovementRow.usuario` |
| `nota` | `string` | `MovementRow.nota` |

### Reglas de validación y de contenido

- **Todo campo de texto es texto plano** y se **escapa a HTML** al generar el documento (`& < > " '`); ningún campo se interpreta como HTML (research D9). Caso de prueba: nota `<img src=x onerror=alert(1)>` se imprime literal.
- El documento **no inventa ni recalcula cifras**: usa exactamente los valores ya resueltos por el store, de modo que pantalla y papel coinciden (FR-001).
- `movimientos.length` se imprime en el encabezado "Movimientos del turno (N)" igual que en pantalla.
- Un campo ausente (`null`/vacío) se imprime como "—" en las celdas de tabla y se omite en las líneas opcionales (observación, cambio entregado), igual que hoy.

### Transiciones de estado

No hay máquina de estados nueva. El ciclo de la impresión (efímero, en el navegador):

```text
[pantalla de reporte] --pulsar "Imprimir / Exportar reporte"-->
  [iframe oculto creado + documento escrito + título fijado]
    --load--> [diálogo de impresión del navegador]
      --imprimir | cancelar | afterprint | 60 s sin evento-->
  [iframe retirado + título de la pestaña restaurado] --> [pantalla de reporte, sin cambios]
```

Invariante (FR-007): en cualquiera de las cuatro salidas, **el DOM de la app queda idéntico al de antes de pulsar** (sin clases en `body`, sin iframes residuales, título original).
