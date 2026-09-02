# Contrato: resumen de una regla en la tarjeta colapsada (FR-001 a FR-005)

**Normativo.** Define el texto exacto que produce `ruleSummaryText(ruleIndex)` en
`pos-heladeria/src/app/modules/promotions/pages/promotions-page.component.ts`, que reemplaza la
implementación actual (líneas 1043-1050). Es distinto, a propósito, del texto de
`ruleConditionPreview` (pantalla de revisión, spec 066) — ver research.md D1.

---

## 1. Entradas

- `type`: `'percent' | 'package_price'`.
- `value`: número (porcentaje 0-100, o precio de paquete en pesos).
- `min_qty`: entero ≥ 1.
- `names`: nombres de las variantes seleccionadas de la regla, en el orden que sea — el
  descriptor los reordena. Viene de `selectedVariantsForRule(ruleIndex).map(v => v.variantName)`.

Reutiliza `setDescriptor(names)` de `promotion-condition.util.ts` (sin modificar): dedup, orden
alfabético sin tildes/mayúsculas, tope de tres + "y N más". Ver
[texto-condicion.md de la spec 066](../../066-promociones-legibles-menu/contracts/texto-condicion.md)
para el algoritmo completo de `setDescriptor`.

## 2. Plantilla de salida

`typeLabel(type)` (sin cambio: "Descuento %" / "Precio de paquete") se sigue anteponiendo en el
*template* del componente (`{{ typeLabel(rule.type) }} - {{ ruleSummaryText(ruleIndex) }}`), **no**
dentro de la función. `ruleSummaryText` devuelve solo lo que va después del guion.

| Caso | Condición | Plantilla |
|---|---|---|
| Conjunto vacío | `names.length === 0` (tras filtrar vacíos) | `Sin productos seleccionados.` |
| `package_price` | cualquier `min_qty` | `Paga {money(value)} llevando {min_qty} {unidad} {descriptor}.` |
| `percent`, `min_qty === 1` | — | `{value}% en {descriptor}.` |
| `percent`, `min_qty > 1` | — | `{value}% llevando {min_qty} {unidad} {descriptor}.` |

- `{unidad}` = `"unidad"` si `min_qty === 1`, si no `"unidades"` (ya existía en el código actual,
  sin cambio).
- `{descriptor}` = el `text` de `setDescriptor(names)` — incluye el conector `"entre "` cuando
  `multiple === true` (idéntico criterio a FR-004 de la spec 066).
- `{money(value)}` usa el mismo formateador que el resto del formulario
  (`formatMoney`/`this.money(...)`).

## 3. Tabla de casos (para el test de aceptación)

| type | value | min_qty | names seleccionados | Salida de `ruleSummaryText` |
|---|---|---|---|---|
| `package_price` | 12000 | 2 | `["Gaseosa - Única"]` | `Paga $ 12.000 llevando 2 unidades Gaseosa - Única.` |
| `package_price` | 12000 | 2 | `["Banana Split Especial - Pequeña"]` | `Paga $ 12.000 llevando 2 unidades Banana Split Especial - Pequeña.` |
| `percent` | 10 | 1 | `["Cono sencillo - Única"]` | `10% en Cono sencillo - Única.` |
| `percent` | 15 | 3 | `["Grande 16oz", "Mediano 12oz", "Pequeño 8oz"]` | `15% llevando 3 unidades entre Grande 16oz, Mediano 12oz y Pequeño 8oz.` |
| `package_price` | 15000 | 2 | 5 nombres distintos | `Paga $ 15.000 llevando 2 unidades entre {3 primeros alfabético} y 2 más.` |
| `package_price` | 12000 | 2 | `[]` (conjunto vacío) | `Sin productos seleccionados.` |

La fila de 5 nombres reutiliza literalmente el ejemplo de `setDescriptor` ya probado por la spec
066 — no se reinventa el corte en tres.

## 4. Fuera de alcance

- El texto de `condition_text` que sirve el backend (listado de administración, menú QR,
  terminal) — sin cambio, ya lo resolvió la spec 066 (A-66).
- `ruleConditionPreview` (pantalla de revisión) — sin cambio, ya nombra variantes desde la spec
  066.
