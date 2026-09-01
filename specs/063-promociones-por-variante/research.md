# Research — Refactorización del módulo de promociones (partición Promoción/Regla)

**Fase 0 de `/speckit-plan`, revisión 2026-09-01.**

> Este `research.md` **reemplaza** la versión anterior (18 decisiones para el modelo plano,
> D1–D18). Esas 18 decisiones siguen vigentes en el código de las ramas de feature — no se
> reabren aquí. Este documento cubre **solo** las decisiones técnicas nuevas que introduce la
> partición `Promoción`/`Regla` (Clarifications, sesión 2026-09-01 de `spec.md`), numeradas
> D-R1…D-R7 para no colisionar con las D1–D18 originales (siguen citables desde
> `contracts/migracion.md` y el código existente).

## D-R1 — `PromotionRule` como tabla propia vs. columna JSONB en `Promotion`

**Decision**: `promotion_rules` es una tabla relacional propia (FK a `promotions`), no una columna
JSONB de tipo `list[dict]` en `promotions`.

**Rationale**: `PromotionVariant` (la tabla puente hacia variantes) necesita una FK relacional
hacia la regla dueña del conjunto — con JSONB no hay a qué apuntar. Además, `_guard_variant_overlap`
y `_guard_package_is_discount` (FR-014, FR-016) necesitan consultar reglas por su propio
`min_qty`/`value`/`type` con `WHERE`s de SQL, no deserializar JSON en Python fila por fila cada vez
que se guarda o activa una promoción — con decenas de promociones por tenant el costo es
irrelevante en cualquiera de los dos enfoques, pero la tabla relacional mantiene la misma forma de
consulta que ya usa el resto del módulo (`select(Promotion).where(...)`, sin un patrón distinto
para esta sola tabla).

**Alternatives considered**:
- *JSONB `rules` en `promotions`*: descartado — obligaría a `PromotionVariant` a apuntar a
  `(promotion_id, rule_index)` en vez de un FK simple, y cualquier validación de FR-016 tendría que
  cargar la promoción completa y deserializar en Python. Rompe el patrón `Mapped`/`mapped_column`
  que usa el resto del dominio.
- *Reusar `promotion_variants` con `type`/`value`/`min_qty` inline (denormalizado, sin tabla de
  regla)*: descartado — dos variantes de la misma regla tendrían el mismo `type`/`value`/`min_qty`
  repetido en cada fila, sin una forma limpia de garantizar que no diverjan, y el conteo "cuántas
  reglas tiene esta promoción" (para el límite por plan, fuera de alcance pero mencionado en Out of
  Scope de spec.md) exigiría un `SELECT DISTINCT` sobre columnas denormalizadas en vez de un
  `COUNT` sobre una tabla propia.

## D-R2 — Migraciones nuevas (`063c`/`063d`) vs. modificar `063a`/`063b` in situ

**Decision**: dos migraciones Alembic **nuevas**, `063c` (aditiva) y `063d` (destructiva),
encadenadas con `down_revision` sobre el head actual (`ba4b6bd573a6` = `063b`). No se edita
ningún archivo de migración ya aplicado.

**Rationale**: decisión explícita del propietario del repositorio (Clarifications, sesión
2026-09-01, Option A): construir sobre las ramas de feature ya existentes, no reescribirlas. Editar
`063a`/`063b` in situ solo tendría sentido si esas migraciones nunca se hubieran aplicado en ningún
entorno — pero ya corrieron en las ramas de feature (aunque esas ramas no estén en `main`, sí
tienen bases de datos de desarrollo/CI con ese estado real) y sus commits ya están en el historial
de `pos-backend`. Reescribirlas exigiría además reescribir los 8 commits que las acompañan
(Principio VI, "no mezclar migración de datos con cambio de cálculo" ya se cumplió una vez para
`063a`/`063b`; volver a mezclarlo sería retroceder esa garantía).

**Alternatives considered**:
- *Squash: reescribir `063a` para incluir `promotion_rules` desde el inicio*: descartado por lo
  anterior — solo sería correcto si se estuviera reescribiendo el historial de la rama de feature
  desde cero, que es exactamente lo que Option A evita.
  Alternativa B (Option C original) — mergear primero el modelo plano a `main` y abrir un spec
  nuevo — habría hecho este squash innecesario también, por la misma razón (nunca se edita una
  migración ya en `main`). No fue la opción elegida.

## D-R3 — Forma del DTO `MenuPromotionAnnouncement.rules[]` en el menú QR (pos-heladeria)

**Decision**: no se cambia la forma de `MenuPromotionAnnouncement` en
`diner.service.ts:27-35` (`{ promotion_id, promotion_name, rules: {text, variant_count, min_qty,
value}[] }`). El backend simplemente empieza a poblar `rules` con **N** elementos por promoción
(uno por cada `PromotionRule` de esa promoción) en vez de siempre 1.

**Rationale**: hallazgo verificado en el código actual — el DTO del menú QR ya modela "una
promoción anuncia una lista de reglas", probablemente como diseño defensivo original de la spec
040/063 aunque el backend plano solo poblara un elemento. Esto reduce el riesgo de este refactor
específicamente en la superficie del menú QR: el contrato entre backend y frontend no cambia de
forma, solo dejan de asumir cardinalidad 1 los lugares que renderizan `rules[0]` en vez de iterar
(a verificar en `public-menu.component.ts` durante la implementación).

**Alternatives considered**: ninguna — es una constatación del código existente, no una decisión
de diseño con alternativas reales.

## D-R4 — Formulario multi-regla: `*ngFor` + `ngModel` indexado vs. `ReactiveFormsModule`/`FormArray`

**Decision**: la sub-lista repetible de reglas dentro de `promotions-page.component.ts` se
implementa con el mismo patrón **template-driven** que ya usa el resto del formulario
(`FormsModule`/`ngModel`, confirmado sin `FormGroup`/`FormArray` en las 900 líneas actuales):
`*ngFor="let rule of form.rules; let i = index; trackBy: trackByIndex"` con
`[(ngModel)]="form.rules[i].value"` (o un objeto de regla referenciado directamente, no por
índice recalculado, para que `trackBy` sea estable al agregar/quitar filas).

**Rationale**: Principio IX (ninguna dependencia nueva) y Principio V (nada de refactor oportunista
más allá de lo que el FR exige) — migrar un formulario de 900 líneas de patrón `ngModel` a
`ReactiveFormsModule`/`FormArray` para ganar una sola lista repetible sería una reescritura mucho
mayor que lo que FR-001/FR-005 piden, y Angular sí soporta arrays repetibles con formularios
template-driven (es menos ergonómico para validación cruzada entre filas, pero FR-001a —variante
repetida entre reglas— se valida igual del lado del cliente con una función TypeScript sobre el
arreglo `form.rules`, no necesita el mecanismo de validadores de `ReactiveFormsModule`).

**Alternatives considered**:
- *`ReactiveFormsModule` + `FormArray<FormGroup>`*: da validación cruzada más idiomática, pero
  exige reescribir el resto del formulario (que sigue con `ngModel`) o mezclar dos patrones de
  formulario en el mismo componente — descartado por Principio V.
- *Componente hijo `<promotion-rule-form>` por fila, con `@Output()` de cambios*: viable y más
  limpio a mediano plazo, pero es una refactorización de estructura de componentes que el FR no
  pide; se deja como nota para una futura limpieza, no parte de este plan.

## D-R5 — Alcance de la reescritura de tests: solo lo que toca `rule_id`/reglas, no todo el módulo

**Decision**: los characterization tests de `cart`/`checkout`/`table_sessions`/`sales` que ya
pasan contra el modelo plano se tocan **solo** donde su aserción depende de la forma de
`applied_promotions` (gana `rule_id`) o del texto de condición (`variant_set_condition_text` ahora
recibe una regla). El resto de esas suites (que no aserta sobre la forma exacta del descuento, solo
sobre el total cobrado) no requiere cambio.

**Rationale**: Principio III exige reescribir un test `CONGELA` solo cuando el comportamiento que
protege efectivamente cambió, citando la razón — no da licencia para tocar un test que sigue
pasando sin motivo. El inventario exacto test por test se completa en
[contracts/migracion.md](./contracts/migracion.md) §"Inventario de tests" durante la
implementación (Principio VIII lo permite explícitamente para el detalle de bajo nivel).

**Alternatives considered**: reescribir preventivamente todas las suites de promociones —
descartado, contradice Principio V y generaría diffs de revisión mucho más grandes de lo que el
cambio real justifica.

## D-R6 — FR-001a (variante no repetida entre reglas de la misma promoción) como guard de aplicación, no `CHECK`/índice parcial

**Decision**: el invariante "dentro de una misma promoción, sus reglas no comparten variante" se
valida en `_guard_variant_overlap` (Python, en la misma función que ya valida el solapamiento entre
promociones distintas, FR-014), no con un `CHECK` o índice parcial de PostgreSQL.

**Rationale**: exactamente el mismo patrón que el módulo ya usa para FR-014 — ese bloqueo tampoco
es un `CHECK` de base de datos (requiere comparar ventanas de tiempo entre filas, no expresable en
un `CHECK` de una sola fila) sino un guard de aplicación que corre antes de `INSERT`/`UPDATE`.
Mantener el invariante nuevo en el mismo lugar y con el mismo mecanismo que el invariante hermano
evita introducir un segundo patrón de validación (índice parcial `WHERE` sobre un `JOIN` con
`promotion_rules.promotion_id`, que PostgreSQL no soporta directamente para unicidad cruzada de
tablas) para un caso que el guard existente ya sabe resolver con una condición adicional (mismo
`promotion_id` → bloquea siempre, sin necesidad de comparar ventanas de tiempo porque son
idénticas por definición, FR-001a).

**Alternatives considered**:
- *Índice único parcial o trigger de PostgreSQL*: técnicamente posible con un trigger `BEFORE
  INSERT/UPDATE`, pero introduce lógica de negocio en la base de datos que el resto del módulo no
  usa en ningún otro punto (todas las demás validaciones de negocio de promociones viven en
  `service.py`) — descartado por consistencia con el patrón ya establecido.

## D-R7 — Incrementos de entrega (G1–J) sobre los A–F ya construidos

> **Corrección `/speckit-analyze` 2026-09-01 (hallazgo F2)**: la versión original de esta decisión
> definía un único "Incremento G" que bundleaba la migración `063c` **y** la reescritura del
> motor en la misma unidad verificable, sin checkpoint intermedio — exactamente lo que Principio
> VI pide evitar ("se evitan cambios masivos que mezclen... migraciones de datos y cambios de
> comportamiento en una misma unidad") y que los Incrementos A/C originales sí habían respetado.
> Se corrige dividiendo G en **G1** (migración + modelo, sin tocar el motor) y **G2** (motor).

**Decision**: 5 incrementos nuevos, verificables por separado, que se apilan sobre los 6 (A–F) ya
completos en las ramas de feature:

- **G1**: migración aditiva `063c` + modelo `PromotionRule`. **No toca el motor** — el CRUD y el
  cálculo de descuento siguen leyendo `promotions.type/value/min_qty` directo, exactamente como
  antes de este refactor. Checkpoint: la suite de characterization tests existente pasa **sin
  editar ninguna línea**, contra la base ya migrada por `063c`. Este checkpoint aísla cualquier
  fallo de la migración (paso de datos, `CHECK`s nuevos) de cualquier fallo del motor, que todavía
  no cambió.
- **G2**: motor (`evaluate_variant_sets` y sus auxiliares: `active_variant_set_rules`,
  `variant_set_condition_text`, `menu_unit_discount`) reescrito para operar sobre
  `PromotionRule` en vez de `Promotion`. Cierra con la suite adaptada (fixtures con
  `add_rule_to_promotion`) en verde, con el CRUD todavía creando promociones de exactamente una
  regla (compatible con los tests existentes).
- **H**: CRUD multi-regla (`create`/`update_shape`/`duplicate`/`serialize_promotion`) +
  `_guard_variant_overlap` con el caso FR-001a + `_guard_package_is_discount` por regla.
- **I**: frontend — interfaces, `promotion.service.ts`, formulario con sub-lista de reglas
  (D-R4), `pos-terminal.store.ts`, `diner.service.ts`/`public-menu.component.ts` (D-R3).
- **J**: migración destructiva `063d` (borra `promotions.type/value/min_qty` y
  `promotion_variants.promotion_id`) + reescritura final de los tests CONGELA que quedaban
  pendientes de la forma exacta de `applied_promotions`.

**Rationale**: mismo criterio que ya se usó para A–F (Principio VI): ningún incremento mezcla el
paso de datos (solo G1) con el cambio de cálculo (solo G2), con el cambio de CRUD (solo H) ni con
el borrado de estructura (solo J). G2 deja el motor funcionando sobre reglas mientras el CRUD
todavía crea promociones con exactamente una regla (compatible con los tests existentes), así que
G2 es verificable de forma aislada antes de que H exponga la creación multi-regla al usuario.

**Alternatives considered**:
- *Colapsar G1+G2 en un solo incremento* (la decisión original, antes de la corrección F2) —
  descartado: mezcla migración de datos con cambio de comportamiento de cálculo en la misma
  unidad verificable, violando el mismo patrón que Principio VI ya exigió para `063a`/`063b`
  (donde la migración —Incremento A— se mantuvo estrictamente separada del cambio de motor
  —Incremento C—).
- *Colapsar G+H en un solo incremento* (motor + CRUD juntos) — descartado, mezclaría cambio de
  esquema/cálculo con cambio de superficie de API en el mismo commit.
