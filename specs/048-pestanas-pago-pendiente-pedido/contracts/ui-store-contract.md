# Contrato de UI/Store: Pestañas para Ver el Pedido Pagado Junto al Pago Pendiente de la Misma Mesa

Esta spec no expone ni consume ninguna API HTTP nueva ni modificada. El "contrato" relevante aquí
es el contrato interno entre `table-sessions.component.ts` y `pos-terminal.store.ts`, para que
`/speckit-tasks` pueda descomponer el trabajo sin ambigüedad sobre quién expone qué.

## Contrato: `pos-terminal.store.ts` → `table-sessions.component.ts` (encabezado y contenido del panel central)

| Miembro | Dirección | Firma | Contrato |
|---|---|---|---|
| `hasPendingAndActiveOrders` (nuevo) | store → componente | `computed<boolean>` | El componente lo lee para decidir si el encabezado pinta las dos pestañas o el texto plano de siempre (FR-001/FR-005) |
| `centralPanelTab` (nuevo) | store ⇄ componente | `signal<'validar-pago' \| 'pedido'>` | El componente lee el valor activo para resaltar el botón correspondiente, y escribe uno nuevo al hacer clic en cualquiera de los dos botones — único punto de escritura |
| `effectiveCentralView` (nuevo) | store → componente | `computed<'validar-pago' \| 'mesa-libre' \| 'pedido'>` | El `@switch` de contenido (líneas 119-149) pasa a leer este computed en vez de `centralState()` — mismo tipo de valor, mismos tres `@case`, cuerpo sin cambios |
| `centralState` (ya existente, sin cambios) | store → componente | `computed<'validar-pago' \| 'mesa-libre' \| 'pedido'>` | Sigue siendo la fuente de verdad sobre qué hay en la mesa; el encabezado lo sigue usando tal cual dentro de la rama `@else` (cuando no hay ambos tipos de pedido a la vez) |
| `pendingOfSelectedTable` (ya existente, sin cambios) | store → componente | `computed<DiningOrder[]>` | Sigue alimentando `[orders]` de `<app-payment-validation-block>` exactamente igual que hoy |
| `reload()` (ya existente, sin cambios) | componente → store | `() => Promise<void>` | Sigue siendo el `(refresh)` de `<app-payment-validation-block>`, sin cambios |

**Nota de compatibilidad**: ningún miembro público existente cambia de firma — `centralState()`
sigue devolviendo exactamente lo mismo que hoy para cualquier consumidor que no sea este componente
(no hay otros, según research.md de spec 047). `payment-validation-block.component.ts`,
`pos-order-panel.component.ts` y `pos-checkout-panel.component.ts` no ganan ni pierden ningún
`@Input`/`@Output` — se montan con las mismas entradas de siempre, solo que ahora pueden coexistir
en el tiempo (uno visible, el otro a un clic de distancia) en vez de excluirse.
