# Data Model: Partición `Promoción` / `Regla`

> **Reemplaza** la versión anterior de este documento (tabla `promotion_variants` nueva,
> `Promotion` con `type`/`value`/`min_qty` propios, retiro de `priority`/`Presentation` — todo eso
> **ya está aplicado** en la rama de feature `refactor/063-promociones-por-variante` de
> `pos-backend`, revisiones `387ef3e638cd` (`063a`) y `ba4b6bd573a6` (`063b`, head actual). Este
> documento cubre **solo** el cambio nuevo: partir `Promotion` en `Promotion` (vigencia + estado) +
> `PromotionRule` (tipo, valor, cantidad mínima, conjunto), con dos migraciones nuevas (`063c`
> aditiva, `063d` destructiva) encima de las ya aplicadas.

Todas las entidades viven en el schema **`tenant`** (por-tenant, vía `@for_each_tenant_schema`).
Las decisiones detrás de cada elección están en [research.md](./research.md) (D-R1…D-R7); este
documento se limita a columnas, restricciones, relaciones, el paso de datos y el rollback
(Principio VIII).

**Resumen del cambio de esquema**:

*Revisión `063c` (aditiva) — deja `promotions.type`/`value`/`min_qty` y
`promotion_variants.promotion_id` intactos, el código de aplicación deja de leerlos/escribirlos en
el mismo incremento (G):*

| Acción | Objeto |
|---|---|
| **Tabla nueva** | `promotion_rules` (hija de `promotions`) |
| **Columna nueva** | `promotion_variants.promotion_rule_id` (FK → `promotion_rules.id`, nullable en esta revisión) |
| **CHECKs nuevos** | `promotion_rules.type` **sin** `CHECK` de valores (ver nota de la tabla `PromotionRule` — Postgres no admite subconsultas en un `CHECK`, así que el escape `OR status='finished'` que sí funciona en `promotions` —porque `status` es una columna de la misma fila— no es replicable aquí sin agregarle a `promotion_rules` una columna de estado que el diseño deliberadamente no tiene); `ck_promotion_rule_value_positive`; `ck_promotion_rule_min_qty`; `ck_promotion_rule_percent_range` |
| **Paso de datos** | Por cada fila de `promotions`: INSERT en `promotion_rules` con su `type`/`value`/`min_qty` (1 regla por promoción, migración 1:1 sin ambigüedad — toda promoción existente ya tiene exactamente una combinación); UPDATE `promotion_variants.promotion_rule_id` = el `id` de esa regla nueva, para las filas cuyo `promotion_id` coincida |

*Revisión `063d` (destructiva) — cuando el código de aplicación ya solo lee/escribe
`promotion_rules`/`promotion_rule_id` (Incremento J):*

| Acción | Objeto |
|---|---|
| **Columnas borradas** | `promotions.type`, `promotions.value`, `promotions.min_qty` |
| **CHECKs borrados** | `ck_promotion_type`, `ck_promotion_value_positive`, `ck_promotion_min_qty`, `ck_promotion_percent_range` (todos en `promotions`, ya redundantes con sus equivalentes en `promotion_rules`) |
| **Columna borrada** | `promotion_variants.promotion_id` (+ su FK + su índice + la `UNIQUE(promotion_id, product_variant_id)` vieja) |
| **Columna endurecida** | `promotion_variants.promotion_rule_id` pasa a `NOT NULL` |
| **UNIQUE nueva** | `(promotion_rule_id, product_variant_id)` en `promotion_variants`, reemplaza la vieja |

Cero cambios de importe en `sales` / `invoices` / `customer_orders` ya emitidas (Principio VII):
el paso de datos de `063c` opera sobre `promotions`/`promotion_variants` (configuración), nunca
sobre una venta o factura.

---

## Entidad nueva: `PromotionRule` (`promotion_rules`)

`app/models/promotion.py` — clase nueva, junto a `Promotion` y `PromotionVariant`. Una fila = "una
combinación (tipo, valor, cantidad mínima) más un conjunto de variantes, dentro de una promoción"
(FR-001a). Es la unidad que hoy porta directamente `Promotion` (`promotion.py:43-114`, verificado
en la rama de feature 2026-09-01) en el modelo plano.

| Columna | Tipo | Nulable | Default | Notas |
|---|---|---|---|---|
| `id` | UUID (PK) | No | `uuid4` | `UUIDPrimaryKeyMixin`, igual que el resto del dominio. |
| `promotion_id` | UUID (FK → `promotions.id`) | No | — | `ondelete="CASCADE"`, indexado. Borrar la promoción borra sus reglas (y, en cascada, sus filas de `promotion_variants`). |
| `type` | String(50) | No | — | Restringido a `{percent, package_price}` **solo en la capa Pydantic** (`PromotionRuleIn.type`), **sin `CHECK` de valores en la base de datos** — ver nota debajo. El paso de datos de `063c` copia el `type` histórico de **toda** promoción existente —incluidas las `Finalizada` con un tipo fuera de `{percent, package_price}` (`combo`/`fixed`/`qty_price`/`qty_price_presentation`)— sin filtrar por `status`; un `CHECK` que solo permitiera los dos tipos vivos bloquearía ese `INSERT` (hallazgo F1, `/speckit-analyze` 2026-09-01). |
| `value` | Numeric(12,2) | No | `0` | Porcentaje (0 < value ≤ 100) o precio total del paquete en COP, según `type` (FR-002, FR-006). Mismo `server_default="0"` que hoy tiene `promotions.value` (permite el paso de datos sin exigir un valor antes de validar). |
| `min_qty` | Integer | No | `1` | Cantidad mínima de unidades elegibles del conjunto de **esta regla** para formar un grupo completo (FR-007). |

> **Por qué `type` no lleva `CHECK`**: `ck_promotion_type` en `promotions` sí puede escapar con `OR status = 'finished'` porque `status` es una columna **de la misma fila** — un `CHECK` de PostgreSQL no admite subconsultas ni referencias a otras tablas (verificado empíricamente: `cannot use subquery in check constraint`). `promotion_rules` no tiene una columna de estado propia por diseño (FR-001: el estado vive en `Promotion`), así que no hay ninguna columna de la misma fila contra la cual escapar. Agregar una columna de estado redundante solo para sostener un `CHECK` violaría Principio V (complejidad no pedida por ningún FR). La restricción a `{percent, package_price}` para escritura nueva vive enteramente en `PromotionRuleIn` (Pydantic, capa de entrada) — mismo criterio que ya usa `PROMOTION_TYPES` en `promotion.py`, que tampoco intenta acotar por `CHECK` los valores históricos que `Promotion.type` puede tener.

**Restricciones**:

| Nombre | Tipo | Definición |
|---|---|---|
| `pk__promotion_rules` | PK | `(id)` |
| `fk__promotion_rules__promotion_id__promotions` | FK | `promotion_id → promotions.id`, `ON DELETE CASCADE` |
| `ck_promotion_rule_value_positive` | CHECK | `value >= 0` (mismo operador que `ck_promotion_value_positive` en `promotions`, no `> 0`) |
| `ck_promotion_rule_min_qty` | CHECK | `min_qty >= 1` |
| `ck_promotion_rule_percent_range` | CHECK | `type <> 'percent' OR value <= 100` |
| `ix__promotion_rules__promotion_id` | INDEX | `(promotion_id)` — `_guard_variant_overlap` y `serialize_promotion` cargan todas las reglas de una promoción por esta columna. |

**Relaciones**:
- `Promotion.rules: list[PromotionRule]`, `cascade="all, delete-orphan"` (reemplaza a
  `Promotion.variants`, que pasa a vivir en `PromotionRule.variants`).
- `PromotionRule.promotion: Promotion` (`back_populates="rules"`) — el motor y `_guard_variant_overlap`
  necesitan subir de una regla a su promoción para resolver la vigencia (FR-001) y para encontrar
  las reglas hermanas (FR-001a).
- `PromotionRule.variants: list[PromotionVariant]`, `cascade="all, delete-orphan"`.

---

## Entidad modificada: `PromotionVariant` (`promotion_variants`)

`app/models/promotion.py:117-149` (verificado en la rama de feature). Ya existe desde `063a`; este
refactor la repunta de `promotion_id` a `promotion_rule_id`.

| Columna | Cambio |
|---|---|
| `id` | Sin cambio. |
| `promotion_id` | **`063c`**: se conserva sin tocar (compatibilidad con el código que aún no migró en el mismo incremento). **`063d`**: se borra, junto con su FK, su índice y la `UNIQUE(promotion_id, product_variant_id)` vieja. |
| `promotion_rule_id` | **Nueva en `063c`** (FK → `promotion_rules.id`, `ON DELETE CASCADE`, nullable). **`063d`**: pasa a `NOT NULL`. |
| `product_variant_id` | Sin cambio (`ondelete="CASCADE"` — FR-011 se conserva igual: una variante eliminada sale del conjunto de toda regla sin fila huérfana). |

**Restricciones nuevas/cambiadas**:

| Nombre | Tipo | Definición | Vigente desde |
|---|---|---|---|
| `fk__promotion_variants__promotion_rule_id__promotion_rules` | FK | `promotion_rule_id → promotion_rules.id`, `ON DELETE CASCADE` | `063c` |
| `uq__promotion_variants__promotion_rule_id__product_variant_id` | UNIQUE | `(promotion_rule_id, product_variant_id)` | `063d` (reemplaza la vieja sobre `promotion_id`) |
| `ix__promotion_variants__promotion_rule_id` | INDEX | `(promotion_rule_id)` | `063c` |

Nota: esta `UNIQUE` solo impide que la **misma** variante se repita **dentro de la misma regla**
(caso sin sentido — listar dos veces la misma variante en un conjunto). El invariante de FR-001a
("una variante no puede pertenecer a dos **reglas distintas** de la **misma** promoción") no es
expresable como `UNIQUE`/`CHECK` sobre esta tabla sola —requeriría conocer `promotion_rules.
promotion_id` desde `promotion_variants`, un `JOIN` que PostgreSQL no soporta en una restricción de
tabla simple— así que se valida en `_guard_variant_overlap` (research.md D-R6), igual que FR-014
entre promociones distintas.

**Relaciones**:
- `PromotionVariant.rule: PromotionRule` (`back_populates="variants"`) — reemplaza a
  `PromotionVariant.promotion`.

---

## Entidad modificada: `Promotion` (`promotions`)

`app/models/promotion.py:43-114` (verificado en la rama de feature).

**Se borra — en `063d` (Incremento J), no en `063c`**, para que el código de aplicación tenga un
incremento entero (G, H, I) para dejar de leer/escribir estas columnas antes de que la revisión
destructiva las borre (mismo patrón que ya usaron `063a`/`063b` con `priority`/`Presentation`):
- Columna `type` (String(50)) y su `ck_promotion_type`.
- Columna `value` (Numeric(12,2)) y su `ck_promotion_value_positive`.
- Columna `min_qty` (Integer) y su `ck_promotion_min_qty`.
- `ck_promotion_percent_range`.

**Se conserva sin cambio**: `id`, `name`, `description`, `status`, `starts_at`, `ends_at`,
`days_of_week`, `start_time`, `end_time`, `closed_by_refactor_at`, `ck_promotion_status`,
`Index("ix_promotions_status_ends_at", "status", "ends_at")`. Ninguna de estas columnas pertenece
al alcance de FR-001a ni de la partición — son exactamente lo que FR-001/FR-012 definen como
"vigencia + estado de la promoción", y ya eran las únicas columnas editables en `Activa`/`Pausada`
antes de este refactor (FR-018 ya excluía `type`/`value`/`min_qty`/conjunto en el modelo plano) —
ese comportamiento no cambia, solo la ubicación física de las columnas que quedan bloqueadas.

**Se modifica**:
- Relación `variants: list[PromotionVariant]` (`promotion.py:90-92`) se **retira** de `Promotion`
  y se agrega a `PromotionRule` (arriba).
- Relación nueva `rules: list[PromotionRule]`, `cascade="all, delete-orphan"`.

---

## Migración y rollback

### `063c_promociones_reglas_aditivo.py`

`down_revision = "ba4b6bd573a6"` (head actual, `063b`). `@for_each_tenant_schema`, con guarda
`_has_table("promotion_rules")` / `_has_column("promotion_variants", "promotion_rule_id")` (mismo
patrón defensivo que `063a`/`063b`, para reintentos idempotentes sobre un tenant ya migrado).

**`upgrade()`**:
1. `CREATE TABLE promotion_rules` con las columnas y `CHECK`s de arriba.
2. `ALTER TABLE promotion_variants ADD COLUMN promotion_rule_id UUID NULL REFERENCES
   promotion_rules(id) ON DELETE CASCADE`.
3. `CREATE INDEX ix__promotion_rules__promotion_id ON promotion_rules(promotion_id)`.
4. `CREATE INDEX ix__promotion_variants__promotion_rule_id ON promotion_variants(promotion_rule_id)`.
5. **Paso de datos** (una sentencia `INSERT ... SELECT` + una `UPDATE ... FROM`, sin cursor fila por
   fila — mismo estilo que el paso de datos de `063a`):
   ```sql
   INSERT INTO promotion_rules (id, promotion_id, type, value, min_qty)
   SELECT gen_random_uuid(), id, type, value, min_qty
   FROM promotions;

   UPDATE promotion_variants pv
   SET promotion_rule_id = pr.id
   FROM promotion_rules pr
   WHERE pr.promotion_id = pv.promotion_id;
   ```
   Correcto porque, en el estado actual (modelo plano), cada `promotions.id` tiene **como mucho
   una** fila resultante en `promotion_rules` (relación 1:1 por construcción de este paso), así que
   el `UPDATE ... FROM` no es ambiguo.

**`downgrade()`**: `DROP` de lo aditivo (índices, columna `promotion_rule_id`, tabla
`promotion_rules`). `promotions.type/value/min_qty` y `promotion_variants.promotion_id` nunca se
tocaron, así que no hay nada que restaurar ahí — el downgrade dis a `promotion_variants` a su
estado de antes de `063c` sin pérdida de información (las filas de `promotion_rules` que se borran
son redundantes con `promotions.type/value/min_qty`, que siguen intactas).

### `063d_promociones_reglas_destructivo.py`

`down_revision = "<063c>"`. `@for_each_tenant_schema` + guarda `_has_column`.

**`upgrade()`**:
1. `ALTER TABLE promotion_variants ALTER COLUMN promotion_rule_id SET NOT NULL` (seguro: `063c` ya
   garantizó que todo `promotion_variants.promotion_rule_id` quedó poblado por el paso de datos, y
   el código de aplicación del Incremento G/H ya solo inserta filas nuevas con `promotion_rule_id`
   directo).
2. `ALTER TABLE promotion_variants DROP CONSTRAINT uq__promotion_variants__promotion_id__product_variant_id`;
   `ADD CONSTRAINT uq__promotion_variants__promotion_rule_id__product_variant_id UNIQUE
   (promotion_rule_id, product_variant_id)`.
3. `ALTER TABLE promotion_variants DROP CONSTRAINT fk__promotion_variants__promotion_id__promotions`;
   `DROP COLUMN promotion_id`; `DROP INDEX ix__promotion_variants__promotion_id` si existe.
4. `ALTER TABLE promotions DROP CONSTRAINT ck_promotion_type, ck_promotion_value_positive,
   ck_promotion_min_qty, ck_promotion_percent_range`; `DROP COLUMN type, value, min_qty`.

**`downgrade()`**: recrea `promotions.type/value/min_qty` (nullable, sin `CHECK` hasta repoblar) y
`promotion_variants.promotion_id` (nullable), y repuebla ambos desde `promotion_rules`/
`promotion_variants.promotion_rule_id` con el paso de datos inverso — **con la misma limitación ya
documentada para `063b`**: si una promoción llegó a tener más de una regla (posible solo después
del Incremento H, cuando el CRUD ya permite crear varias), el downgrade no puede "aplanarlas" de
vuelta a una sola fila de `promotions.type/value/min_qty` sin perder información; en ese caso el
downgrade recrea la estructura vacía (nullable) y deja un aviso en el log de migración, igual que
el criterio ya aceptado para el downgrade de `063b` (recrea estructura, no garantiza datos
equivalentes cuando la operación no es reversible sin pérdida).

---

## Reglas de validación (resumen, detalle en contracts/)

- **FR-001**: una promoción sin ninguna regla no es un estado alcanzable vía la API — `create`
  exige `rules` con al menos 1 elemento (igual que hoy `variant_ids` exige al menos 1).
- **FR-001a**: `_guard_variant_overlap` bloquea si una variante aparece en el conjunto de dos
  reglas de la promoción que se está guardando, **antes** de comparar contra otras promociones
  (research.md D-R6).
- **FR-014/FR-014a**: entre reglas de promociones distintas, sin cambio de criterio — variante
  compartida + intersección de fecha/días/horas de sus respectivas promociones.
- **FR-016**: por regla — `value >= min_qty × precio de la variante más barata del conjunto de esa
  regla` bloquea el guardado, nombrando la regla.
- **FR-018**: en `Activa`/`Pausada`, la sub-lista completa de reglas queda bloqueada (agregar,
  quitar, editar cualquier campo de cualquier regla); solo los campos de `Promotion` (vigencia)
  siguen editables.

## Diagrama de relaciones (textual)

```
Promotion (1) ──< rules >── (N) PromotionRule (1) ──< variants >── (N) PromotionVariant (N) >── (1) ProductVariant
    │                                                                    ▲
    └── vigencia (starts_at/ends_at/days_of_week/start_time/end_time)   │
        + estado (status)                                    conjunto elegible de ESA regla
```
