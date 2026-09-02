# Contrato: texto de condición en lenguaje llano (FR-001 a FR-006)

**Normativo.** Este documento define el único algoritmo que produce el texto de condición de una
regla. Lo implementan **dos** veces —una en Python (fuente única para las tres superficies) y una
en TypeScript (solo la vista previa del formulario, porque describe variantes todavía no
guardadas, FR-018)— y las dos implementaciones deben coincidir carácter por carácter (SC-005).

---

## 1. Firma (backend)

```python
# app/api/v1/promotions/service.py

def variant_display_names(db: Session, variant_ids: Iterable[UUID]) -> dict[UUID, str]:
    """`{product_variant_id: nombre utilizable}` en UNA consulta.

    El nombre utilizable es `variant.name` y, si queda vacío al recortar,
    `product.name`. Una variante sin ninguno de los dos NO aparece en el mapa
    (no aporta nombre, FR-006)."""


def variant_set_condition_text(rule: PromotionRule, names: Mapping[UUID, str]) -> str | None:
    """Condición en lenguaje llano, español de Colombia (FR-004).

    `names` es OBLIGATORIO (research.md D-1): un call site que lo olvide debe
    fallar, no degradar en silencio al texto por conteo y romper SC-005."""
```

`variant_set_condition_text` **conserva** su contrato actual en un punto: devuelve `None` para una
regla cuyo `type` no está en `LIVE_TYPES` (histórica, migrada de una promoción `finished`). Esa
guarda va **primero**, antes de mirar nombres.

### Call sites (los dos únicos)

| Fichero | Origen del mapa `names` |
|---|---|
| `menu/router.py:206` (`_build_menu_promotions`) | `variant_display_names(db, …)` sobre la unión de los conjuntos de las reglas vigentes. Una consulta por llamada. |
| `promotions/service.py:685` (`_serialize_rule`) | Construido del `by_id` que `serialize_promotion` **ya tiene** (`:696-701`). **Cero consultas nuevas.** |

---

## 2. El descriptor del conjunto (FR-002, FR-003)

### 2.1 Clave de ordenación

```python
def _sort_key(name: str) -> str:
    d = unicodedata.normalize("NFD", name)
    return "".join(c for c in d if not unicodedata.combining(c)).casefold()
```

```typescript
function sortKey(name: string): string {
  return name.normalize('NFD').replace(/\p{M}/gu, '').toLowerCase();
}
```

Se ordena por la clave; **se muestra el nombre original**. Ante dos nombres con la misma clave
(`"Pequeño"` y `"pequeño"`), la deduplicación es por nombre mostrado, así que ambos sobreviven y
su orden relativo lo decide el desempate por el nombre original — determinista.

### 2.2 Construcción

1. Para cada `PromotionVariant` del conjunto, tomar `names[product_variant_id]`; omitir las que no
   estén en el mapa.
2. Recortar espacios, descartar vacíos.
3. **Deduplicar** conservando el nombre mostrado (FR-003).
4. Ordenar por `_sort_key`, desempate por el nombre original.
5. Según cuántos nombres distintos (`D`) queden:

| `D` | Descriptor | `multiple` |
|---|---|---|
| 0 | — → **respaldo por conteo**, ver §4 | — |
| 1 | `Pequeño 8oz` | `false` |
| 2 | `Pequeño 8oz y Mediano 12oz` | `true` |
| 3 | `Pequeño 8oz, Mediano 12oz y Grande 16oz` | `true` |
| > 3 | `Pequeño 8oz, Mediano 12oz, Grande 16oz y {D-3} más` | `true` |

Con `D > 3` se listan **los tres primeros del orden** y se resume el resto; `N = D - 3` cuenta
**nombres distintos** restantes, no variantes.

---

## 3. Los cuatro textos (FR-004)

Con `n = rule.min_qty`, `d` = descriptor de §2, `e = "entre "` si `multiple` y `""` si no:

| Tipo | `n` | Texto |
|---|---|---|
| `package_price` | `> 1` | `Llevando {n} {e}{d} pagas {valor}` |
| `package_price` | `= 1` | `Cada {e}{d} a {valor}` |
| `percent` | `= 1` | `{pct}% en {d}` |
| `percent` | `> 1` | `{pct}% llevando {n} {e}{d}` |

Nótese que la variante `percent` con `n = 1` **no lleva** `e`: FR-004 solo lo define para las
otras tres.

### Formatos (FR-005, sin cambio respecto de hoy)

- **Importe**: `_money()` — `"$" + f"{int(value):,}".replace(",", ".")` → `$12.000`.
- **Porcentaje**: sin ceros decimales sobrantes y con **punto** decimal → `10%`, `12.5%`.
  En Python lo produce el código actual: `f"{value:f}"` + `rstrip("0").rstrip(".")`.
  **Se conserva tal cual** — cambiar el separador decimal del porcentaje sería un cambio de
  comportamiento visible fuera del alcance de A-66 (Principio V). FR-005 lo fija así de forma
  explícita, y el de TypeScript debe igualarlo (`String(value)`, no `toLocaleString('es-CO')`).

---

## 4. Respaldo y ausencia de texto (FR-006)

| Situación | Resultado |
|---|---|
| `rule.type not in LIVE_TYPES` (histórica) | `None` — la regla no se anuncia, no genera insignia y no produce información en el modal. **Guarda primero.** |
| Ninguna variante del conjunto aporta nombre | Descriptor **por conteo**, el texto que existe hoy: `de estas {n} variantes` / `estas {n} variantes`, con `n = len(rule.variants)`. |
| Al menos una aporta nombre | Descriptor por nombres (§2). Las que no aportan simplemente no figuran. |

Textos exactos del respaldo (idénticos a producción):

```text
Llevando {min_qty} de estas {n} variantes pagas {valor}
Cada una de estas {n} variantes a {valor}
{pct}% en estas {n} variantes
{pct}% llevando {min_qty} de estas {n} variantes
```

---

## 5. Tabla de casos (ejercitable por los tests de los dos lados)

| # | Tipo | `min_qty` | Valor | Nombres del conjunto | Texto esperado |
|---|---|---|---|---|---|
| 1 | `package_price` | 2 | 12000 | 8 × `Pequeño 8oz` | `Llevando 2 Pequeño 8oz pagas $12.000` |
| 2 | `package_price` | 2 | 12000 | 1 × `Pequeño 8oz` | `Llevando 2 Pequeño 8oz pagas $12.000` |
| 3 | `package_price` | 2 | 15000 | `Pequeño 8oz`, `Mediano 12oz`, `Grande 16oz` | `Llevando 2 entre Grande 16oz, Mediano 12oz y Pequeño 8oz pagas $15.000` |
| 4 | `package_price` | 2 | 15000 | 5 nombres distintos | `Llevando 2 entre {1.º}, {2.º}, {3.º} y 2 más pagas $15.000` |
| 5 | `percent` | 1 | 10 | 8 × `Pequeño 8oz` | `10% en Pequeño 8oz` |
| 6 | `percent` | 3 | 15 | 2 nombres distintos | `15% llevando 3 entre {1.º} y {2.º}` |
| 7 | `package_price` | 1 | 6000 | 1 × `Pequeño 8oz` | `Cada Pequeño 8oz a $6.000` |
| 8 | `percent` | 1 | 10 | todas sin nombre de variante ni de producto | `10% en estas 3 variantes` |
| 9 | `combo` (histórica) | — | — | — | `None` |
| 10 | `percent` | 1 | 12.5 | 1 × `Pequeño 8oz` | `12.5% en Pequeño 8oz` |

> **Ojo con el caso 3**: el orden alfabético manda sobre el orden en que el administrador
> seleccionó las variantes, así que `Grande 16oz` va primero aunque sea el tamaño más grande.
> Los ejemplos de la spec (CA3, CA4 y FR-002) ya enumeran alfabéticamente; los tests deben
> afirmar ese orden, nunca el de selección ni el de tamaño.

---

## 6. Vista previa del formulario (FR-018)

`promotions-page.component.ts` no puede consumir el texto del backend: las variantes seleccionadas
todavía no están guardadas. Replica §2 y §3 con los nombres que ya tiene en memoria.

**Fichero nuevo**: `src/app/modules/promotions/services/promotion-condition.util.ts`

```typescript
export interface ConditionRule {
  type: 'percent' | 'package_price';
  value: number;
  min_qty: number;
}

/** Descriptor de §2 a partir de los nombres ya resueltos. `null` si ninguno sirve. */
export function setDescriptor(names: string[]): { text: string; multiple: boolean } | null;

/** Texto de §3, o el respaldo por conteo de §4 cuando `setDescriptor` da `null`. */
export function conditionText(rule: ConditionRule, names: string[], variantCount: number): string;
```

El nombre que entra por `names` es `${productName} - ${variantName}`… **no**: es el
`variantName` solo, y `productName` cuando el primero está vacío — el mismo criterio de
`variant_display_names`. El formulario ya tiene los dos campos en `CatalogVariant`
(`productName` / `variantName`, `promotions-page.component.ts:57-58`) y los resuelve con
`selectedVariantsForRule(ruleIndex)` (`:884`).

**Cambio de firma**: `ruleConditionPreview(rule)` → `ruleConditionPreview(ruleIndex: number)`, para
poder pedir las variantes seleccionadas de esa regla. La plantilla ya tiene `$index` en ese
alcance (`:554` está dentro del mismo `@for` que usa `selectedVariantsForRule($index)` en `:556`).
El test `promotions-page.component.spec.ts:56` se actualiza a la firma nueva.

---

## 7. Compatibilidad

- **Aditivo en la respuesta**: `condition_text` y `MenuPromotionRule.text` conservan su nombre,
  su tipo y su nulabilidad. Cambia el **contenido**, que es exactamente lo que A-66 autoriza.
- `MenuPromotionRule.variant_count` **se conserva** aunque el texto ya no lo mencione: es un campo
  publicado de la spec 063 y retirarlo sería un cambio de contrato fuera del alcance de esta spec
  (Principio V). Simplemente deja de tener reflejo en el texto.
