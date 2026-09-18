# Contrato: Aplicación masiva de una regla a los productos ya seleccionados (bug 3)

Cubre spec.md FR-016 a FR-020. Cita anomalía **A-77** (research.md D7) — debe existir antes de
mergear. **Sin cambio de backend**: `PromotionRule`/`PromotionVariant` ya soportan N reglas por
promoción, cada una con su propia lista de variantes — el cambio es 100% de cómo el frontend arma
el payload.

## Comportamiento actual (a reemplazar)

`addRuleRow()` / `resolvedVariantIdsForLabel` (`promotions-page.component.ts:1621-1706`): al
definir una regla para una presentación que coincide con varios productos del Paso 1, genera **una
sola** `PromotionRuleForm` cuyo `variantIds` es la unión de las variantes de todos los productos
coincidentes.

## Comportamiento nuevo

1. Al definir una regla (presentación + unidades mínimas + precio) que coincide con más de un
   producto ya seleccionado en el Paso 1, el sistema muestra una lista de esos productos
   coincidentes, cada uno con una casilla de verificación, **todas premarcadas**.
2. El administrador puede desmarcar los que no quiere incluir, antes de confirmar.
3. Al confirmar, `addRuleRow()` genera **una `PromotionRuleForm` independiente por cada producto
   que quedó marcado** — cada una con `variantIds = [variante de ese producto para la
   presentación elegida]`, mismo `type`/`min_qty`/`price` que la regla definida una sola vez.
4. Un producto candidato sin ninguna variante cuyo nombre coincida con la presentación elegida
   **no aparece como opción marcable** (o aparece deshabilitado con la razón) — no se le genera
   fila, sea cual sea el estado de su casilla (no existe casilla para él).
5. Cada `PromotionRuleForm` generada queda en la tabla de reglas de la pantalla de configuración
   como cualquier otra, editable/eliminable de forma individual sin afectar las demás (FR-019).
6. Las validaciones ya vigentes por fila (FR-025/FR-026 de spec 083: mínimo 2 unidades, precio
   menor a la suma de precios regulares) se evalúan **por fila**, usando el precio regular propio
   de la variante de cada producto (spec.md FR-020) — sin cambio de esas funciones de validación,
   solo se invocan una vez por fila generada en vez de una vez por la regla combinada.

## Interacción con la exclusividad de producto (contrato separado)

Ver [exclusividad-producto-promociones.md](./exclusividad-producto-promociones.md). Un producto ya
excluido del Paso 1 por esa guarda nunca llega a ser candidato de esta aplicación masiva (se filtra
antes, en la selección de productos).

## Casos de aceptación cubiertos

- 3 productos seleccionados comparten "Presentación única"; se define la regla una vez → aparecen
  3 checkboxes premarcados; al confirmar sin desmarcar ninguno, se generan 3 `PromotionRuleForm`.
- Se desmarca 1 de los 3 antes de confirmar → se generan solo 2 filas.
- De los 3 seleccionados, 1 no tiene esa variante → no aparece como opción marcable, se generan
  como mucho 2 filas (de los que sí tienen la variante y quedaron marcados).
- Tras generarse 3 filas, se edita el precio de una → solo esa fila cambia, las otras 2 quedan
  igual.
- Se elimina 1 de las 3 filas generadas → las otras 2 permanecen, cada una sigue siendo una
  `PromotionRule` independiente al guardar.
