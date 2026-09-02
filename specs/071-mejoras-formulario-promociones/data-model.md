# Phase 1 — Data Model

**Spec**: [spec.md](./spec.md) · **Plan**: [plan.md](./plan.md) · **Fecha**: 2026-09-02

## Resumen: cero cambios de esquema

**Esta feature no crea, modifica ni elimina ninguna tabla, columna, restricción, índice ni
relación de PostgreSQL. No lleva migración de Alembic. No tiene estrategia de rollback de datos
porque no toca datos.** El Principio VIII no se activa.

El modelo persistido sigue siendo exactamente el de la spec 063:

```text
Promotion (vigencia + estado: draft/active/paused/finished)
└── PromotionRule (type, value, min_qty)
    └── PromotionVariant (product_variant_id)
        └── ProductVariant (name, price) ── Product (name)
```

Lo único que cambia es **qué transición de estado permite reemplazar la lista de `PromotionRule`
de una promoción** (`update_shape`, ver [contracts/edicion-en-pausada.md](./contracts/edicion-en-pausada.md)
§1) y **qué campos del formulario deja tocar el cliente antes de enviar ese reemplazo** — ninguno
de los dos es un cambio de esquema.

---

## Único tipo nuevo: `PromotionRuleForm.isExisting`

El único campo nuevo de esta spec vive en el **formulario del frontend**, no en el backend ni en
la base de datos:

```typescript
// pos-heladeria/src/app/modules/promotions/interfaces/promotion.interface.ts
export interface PromotionRuleForm {
  type: PromotionType;
  value: number;
  min_qty: number;
  variantIds: string[];
  /** NUEVO (spec 071): `true` si esta fila vino de `openEdit()` (la regla ya
   *  existía cuando se abrió el formulario); `false` si se agregó con
   *  "+ Agregar regla" en esta misma sesión de edición. Gobierna
   *  `canEditRuleTypeValue()` (FR-015) — nunca viaja al backend, `toRules()`
   *  arma el payload campo por campo. */
  isExisting: boolean;
}
```

| Campo | Tipo | Origen | Vida |
|---|---|---|---|
| `isExisting` | `boolean` | `true` al mapear `p.rules` en `openEdit(p)`; `false` en `emptyRule()` | Solo mientras dura la sesión del formulario en el navegador. Se descarta al guardar — la próxima vez que se abra la promoción para editar, todas sus reglas vuelven a marcarse `isExisting: true` porque ya existen en el backend. |

**Reglas de validación**: ninguna nueva. `isExisting` no participa de `formValid()` ni de ninguna
guarda de envío — solo decide qué inputs quedan habilitados en pantalla
([contracts/edicion-en-pausada.md](./contracts/edicion-en-pausada.md) §2).

**Compatibilidad con datos existentes**: total. Una promoción cargada desde el backend antes de
este cambio tiene exactamente los mismos campos por regla (`type`, `value`, `min_qty`,
`variant_ids`, `condition_text`); `isExisting` se calcula en el cliente al mapear la respuesta,
nunca se lee del backend.

**Estrategia de rollback**: revertir el commit del frontend. Ningún dato persistido depende de
este campo.

---

## Entidades ya existentes que esta spec solo lee o reordena

- **`Promotion.status`**: sin campo nuevo. Se lee para decidir `isPaused` (nuevo *computed*,
  mismo patrón que `isDraft`/`isReadOnly` ya existentes) y para la condición de `update_shape`
  (backend).
- **`form.rules` (arreglo)**: FR-012 cambia **dónde se inserta** un elemento nuevo (`unshift` en
  vez de `push`), no su forma. `ruleFilters` (arreglo paralelo, cliente-only, nunca se guarda)
  se mueve con la misma operación para mantener el índice alineado.
- **`PromotionVariant` / `ProductVariant`**: sin cambio; se siguen leyendo tal cual para nombrar
  el conjunto (`setDescriptor`, reutilizado de la spec 066) y para poblar
  `selectedVariantsForRule` / `searchResultsForRule`
  ([contracts/busqueda-y-seleccion.md](./contracts/busqueda-y-seleccion.md)).
