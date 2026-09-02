# Contrato: edición de reglas de una promoción `Pausada` (FR-012 a FR-018)

**Normativo.** Documenta el único cambio de comportamiento en producción de esta spec (A-69,
[registro-de-anomalias.md](../../000-reconocimiento/registro-de-anomalias.md)) y la mecánica de
orden de FR-012.

---

## 1. Backend — `PATCH /promotions/{id}/shape`

**Antes** (`service.update_shape`,
[service.py:757-777](../../../../pos-backend/app/api/v1/promotions/service.py#L757-L777)):

```python
if promo.status != "draft":
    raise HTTPException(status.HTTP_409_CONFLICT,
        "Solo una promoción en borrador puede cambiar sus reglas. "
        "Duplícala, edita la copia y finaliza la original.")
```

**Después**:

```python
if promo.status not in ("draft", "paused"):
    raise HTTPException(status.HTTP_409_CONFLICT,
        "Solo una promoción en borrador o pausada puede cambiar sus reglas. "
        "Actívala y páusala para editarla, o duplícala.")
```

Nada más cambia en el cuerpo del método: `_guard_no_shared_variants_within_payload`,
`_add_rules`/`_apply_variant_set`, `_guard_package_is_discount` y `_guard_variant_overlap` corren
exactamente igual que hoy, sobre la lista completa de reglas que manda el payload (reemplazo
total, no upsert — ver research.md D3/D4). El router
([router.py:99-101](../../../../pos-backend/app/api/v1/promotions/router.py#L99-L101)) actualiza
su `summary` de `"solo en borrador"` a `"borrador o pausada"`.

**Tabla de transición** (`Promotion.status` al momento de llamar `PATCH .../shape`):

| Estado | ¿Permite reemplazar la lista de reglas? |
|---|---|
| `draft` | Sí (sin cambio). |
| `paused` | **Sí — nuevo en esta spec (FR-014).** |
| `active` | No — 409 (sin cambio, FR-013). |
| `finished` | No — 409 (sin cambio; además es terminal). |

## 2. Frontend — dos permisos, no uno

`canEditShape()` se reemplaza por dos funciones (research.md D3):

```typescript
canEditRuleSet(): boolean {
  return !this.isReadOnly() && (this.isDraft() || this.isPaused());
}

canEditRuleTypeValue(rule: PromotionRuleForm): boolean {
  return !this.isReadOnly() && (this.isDraft() || (this.isPaused() && !rule.isExisting));
}
```

`isPaused = computed(() => this.editingStatus() === 'paused')` (nuevo, mismo patrón que
`isDraft`/`isReadOnly`).

`PromotionRuleForm.isExisting: boolean` (nuevo, cliente-only, no viaja al payload — `toRules()` en
`promotion.service.ts` ya arma el objeto campo por campo):

- `emptyRule()` → `isExisting: false` (toda regla agregada con "+ Agregar regla" es nueva).
- `openEdit(p)` → cada regla mapeada de `p.rules` → `isExisting: true`.

### Matriz de qué gobierna cada permiso

| Control de la UI | Permiso | `Activa` | `Pausada`, regla ya existía | `Pausada`, regla nueva de esta sesión | `Borrador` |
|---|---|---|---|---|---|
| "+ Agregar regla" / "Quitar regla" | `canEditRuleSet()` | ⛔ | ✅ | ✅ | ✅ |
| Casillas del conjunto de variantes | `canEditRuleSet()` | ⛔ | ✅ | ✅ | ✅ |
| Botones de tipo | `canEditRuleTypeValue(rule)` | ⛔ | ⛔ | ✅ | ✅ |
| Input de valor | `canEditRuleTypeValue(rule)` | ⛔ | ⛔ | ✅ | ✅ |
| Input de cantidad mínima | `canEditRuleTypeValue(rule)` | ⛔ | ⛔ | ✅ | ✅ |

`removeRule(index)` conserva su guarda existente `this.form.rules.length > 1` (además de exigir
`canEditRuleSet()`) — es lo que ya implementa FR-018 en el cliente; el `min_length=1` de
`PromotionShapeUpdate.rules` en el backend es el cinturón de seguridad del lado del servidor, sin
cambio.

`save()` (líneas 921-944) llama a `updateShape` cuando `this.isDraft() || this.isPaused()`, no
solo cuando `isDraft()` — si no, una edición del conjunto hecha en `Pausada` nunca llegaría al
backend.

## 3. Orden del listado de reglas (FR-012)

`addRule()` ([línea 814](../../../../pos-heladeria/src/app/modules/promotions/pages/promotions-page.component.ts#L814)):
cambia `this.form.rules.push(emptyRule())` por `this.form.rules.unshift(emptyRule())`, y
`this.ruleFilters.push(...)` por `this.ruleFilters.unshift({ category: '', text: '' })` —
`ruleFilters` es un arreglo paralelo a `form.rules` por índice, así que debe moverse igual.
`expandedRuleIndex.set(this.form.rules.length - 1)` pasa a `expandedRuleIndex.set(0)` (la regla
nueva, ahora en la posición 0, es la que se expande).

Sin cambio en `removeRule` más allá de lo ya descrito en §2: quitar una regla no reordena las
demás.

## 4. Casos que no cambian (para el test de no-regresión)

- `Activa` sigue bloqueando **todo**: agregar/quitar reglas, tipo, valor, cantidad mínima y
  conjunto — `test_ca2_cambiar_reglas_de_una_activa_bloquea` y el test de Angular "FR-018: en una
  promoción activa las reglas no son editables" seguirán en verde sin tocarlos.
- `Finalizada` sigue sin ninguna edición (terminal).
- `Borrador` sigue completamente editable, sin ninguna de las dos restricciones nuevas.
