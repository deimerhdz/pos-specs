# Quickstart: validar la partición `Promoción`/`Regla`

> **Reemplaza** la versión de este quickstart escrita para el modelo plano — esa migración
> (`063a`/`063b`) y su verificación **ya se hicieron** en la rama de feature de `pos-backend`
> (verificado 2026-09-01: suite completa en verde, migraciones aplicadas). Esta guía **empieza
> desde ese estado ya construido** como línea base y valida solo el delta de la partición
> `Promoción`/`Regla`. No repite firmas ni columnas ya detalladas en
> [data-model.md](./data-model.md) y [contracts/](./contracts/) — solo enlaza a ellas.

**Prerequisitos**: rama `refactor/063-promociones-por-variante` de `pos-backend` (no `main`), venv
activo. Rama `feature/063-promociones-por-variante` de `pos-heladeria` (no `main`).

```bash
cd ../pos-backend
source env/bin/activate
```

Characterization tests sobre SQLite en memoria (sin ejecutar la migración
`@for_each_tenant_schema`). Las migraciones nuevas (`063c`/`063d`) y su paso de datos se prueban
contra PostgreSQL real.

---

## Paso 0 — Línea base: la suite del modelo plano sigue en verde

```bash
python -m unittest discover -s app/characterization_tests -p 'test_*.py' -v
python app/scripts/test_promotions_rules.py
```

**Resultado esperado**: todo en verde, **sin ningún cambio de código todavía** — esto confirma que
se parte del estado real de la rama de feature (Option A, 2026-09-01), no de una hipótesis. Fijar
estos resultados como línea base para comparar al final del Paso 4.

---

## Paso 1 — Migración `063c` (aditiva), contra PostgreSQL real

Sembrar un tenant con al menos 3 promociones existentes (una `Activa` de tipo `percent`, una
`Activa` de tipo `package_price`, una `Finalizada` con `closed_by_refactor_at` no nulo de la
migración `063a` **y `type` legado** —p. ej. `combo` o `fixed`, no `percent`/`package_price` —
para ejercitar que `063c` no falla al copiar un `type` histórico, hallazgo F1 de
`/speckit-analyze` 2026-09-01) y al menos una `Sale`/`Invoice` con `applied_promotions` ya
escrito.

```bash
alembic upgrade head        # aplica 063c: crea promotion_rules; agrega
                             # promotion_variants.promotion_rule_id (nullable); PASO DE DATOS:
                             # una regla por cada promoción existente, copiando type/value/min_qty;
                             # repunta promotion_variants. NO borra promotions.type/value/min_qty
                             # ni promotion_variants.promotion_id — el motor viejo (que aún no se
                             # tocó en este paso) sigue funcionando sobre las columnas viejas.
alembic downgrade -1        # revierte 063c sin error (columnas viejas nunca se tocaron)
alembic upgrade head
```

**Verificación 1** (contra PostgreSQL real):

1. Cada promoción existente tiene ahora **exactamente una** fila en `promotion_rules`, con el
   mismo `type`/`value`/`min_qty` que tenía `promotions` antes de la migración — **incluida** la
   `Finalizada` de `type` legado: el `INSERT` no falla porque `promotion_rules.type` no lleva
   `CHECK` de valores (hallazgo F1; la restricción a `{percent, package_price}` para escritura
   nueva vive solo en el schema Pydantic de entrada, `PromotionRuleIn`).
2. `promotion_variants.promotion_rule_id` apunta a esa regla nueva, para todas las filas que antes
   apuntaban a `promotion_id` de esa promoción — el conjunto de variantes **no cambia**, solo gana
   un nivel de indirección.
3. `promotions.type`/`value`/`min_qty` **siguen existiendo** con su valor original —
   `promotion_variants.promotion_id` también — nada se borró todavía (Incremento G1 es 100%
   aditivo).
4. La `Sale`/`Invoice` sembrada: `discount`, `total`, `applied_promotions` **sin cambio** — el
   paso de datos no tocó ninguna tabla de ventas (Principio VII).

**Verificación de regresión 1:1 — Checkpoint G1** (contracts/migracion.md §4, tasks.md
"Checkpoint G1"): con el motor **todavía** leyendo `promotions.type/value/min_qty` directo (el
Incremento G2, que lo reescribe, todavía no empezó), correr de nuevo el Paso 0 — debe seguir en
verde exacto, byte a byte en los totales esperados, **sin editar ninguna línea de test**. Esto
aísla cualquier fallo de la migración (paso de datos, `CHECK`s) de cualquier fallo del motor, que
se introduce recién en el Incremento G2.

---

## Incremento G2 — Motor sobre reglas

Tras reescribir `evaluate_variant_sets` y sus auxiliares para leer `promotion_rules`
([contracts/motor-y-persistencia.md](./contracts/motor-y-persistencia.md)), con el CRUD **todavía**
sin exponer creación multi-regla (cada promoción sigue teniendo exactamente 1 regla, la que generó
el paso de datos de `063c`):

```bash
python -m unittest app.characterization_tests.test_promotions_service -v
python -m unittest app.characterization_tests.test_orders_checkout -v
python -m unittest app.characterization_tests.test_cart_service -v
```

**Resultado esperado**: los mismos totales que el Paso 0 (una regla produce exactamente el mismo
descuento que producía la promoción plana equivalente) — la diferencia observable es solo que
`applied_promotions` ahora trae `rule_id` por entrada.

---

## Incremento H — CRUD multi-regla y FR-001a

Archivo: `test_promotions_rules_admin.py` (reescrito, [contracts/administracion-promociones.md](./contracts/administracion-promociones.md)).

1. `POST /promotions` con `rules: [6 elementos]` — cada uno tipo `package_price`, su propio
   `value`/`min_qty`/`variant_ids` (caso Springfield: Pequeños $12.000, Medianos $17.000, Grandes
   $22.000, Extra grandes $27.000, Baldes $31.000, Litros $41.000, cada regla con el conjunto con
   licor de su tamaño), vigencia común `daysOfWeek={lunes,martes,miércoles,jueves}` → **201**;
   `status='draft'`; `rules[]` en la respuesta con 6 elementos, cada uno con su `condition_text` y
   sus `variants[]` con `unit_price`. **Una sola llamada, no seis** (User Story 1, creación por
   lote).
2. Mismo payload, pero la regla de Pequeños y la de Medianos comparten una variante por error →
   **409** FR-001a, nombrando las dos reglas en conflicto — antes de siquiera intentar comparar
   contra otras promociones.
3. `PATCH /promotions/{id}/status {"status":"active"}` sobre la promoción del punto 1 → **200**;
   las 6 reglas quedan activas con la misma vigencia.
4. `PATCH /promotions/{id}/status {"status":"paused"}` → **200**; las 6 reglas dejan de aplicar
   descuento con **una sola llamada** (User Story 5, mantenimiento por lote) — verificar cobrando
   un pedido que antes recibía el descuento de cualquiera de las 6 y confirmar que ya no lo recibe.
5. `PATCH /promotions/{id}` con `ends_at` distinto sobre la promoción `Activa` → **200**; las 6
   reglas heredan la nueva fecha de fin sin tocarlas una por una.
6. `PATCH /promotions/{id}/shape` con un `rules` que quita una de las 6 → **409** (promoción
   `Activa`, FR-018: la sección de reglas está bloqueada fuera de `Borrador`).
7. `POST /promotions/{id}/duplicate` sobre una promoción `Activa` con 6 reglas → copia en
   `Borrador` con las 6 reglas idénticas (tipo/valor/`min_qty`/conjunto de cada una) y la misma
   vigencia, nombre distinto.

```bash
python -m unittest app.characterization_tests.test_promotions_rules_admin -v
```

---

## Incremento I — Frontend: formulario multi-regla y superficies de consumo

Verificación manual (`ng serve`, admin → Promociones → "＋ Nueva promoción"):

1. El formulario muestra un bloque de vigencia (fechas, días, horario) y una **lista repetible de
   reglas**, cada una con su propio tipo/valor/cantidad mínima/selector de conjunto — botones
   "agregar regla" / "quitar regla".
2. Agregar las 6 reglas del caso Springfield en la misma sesión del formulario, ver el resumen por
   regla (FR-005), guardar en una sola acción.
3. Intentar que dos reglas del formulario compartan una variante → error de validación del cliente
   **antes** de enviar (feedback inmediato), y si se fuerza el envío, el 409 de FR-001a del
   backend.
4. Abrir una promoción `Activa` con varias reglas → toda la sección de reglas es de solo lectura;
   solo vigencia editable.
5. Pausar esa promoción desde el listado → las 6 reglas dejan de aplicar descuento con una sola
   acción (verificar en la terminal: el badge de descuento desaparece de los 6 tramos).

```bash
cd ../pos-heladeria
ng build
```

Terminal de staff y menú QR:

6. Con la promoción de 6 reglas `Activa`, en la terminal cada una de las variantes de cada tramo
   muestra su propia condición ("Llevando 2 pagas $X") — `pos-terminal.store.ts`
   `productDiscountBadge()` cruzando `promo.rules[].variants` (research.md D-R3/D-R4).
7. En el menú QR (`GET /menu/promotions`), el anuncio de esa promoción trae **6 elementos** en
   `rules[]` — uno por tramo de precio — sin ningún cambio de tipo en `diner.service.ts`
   (`MenuPromotionAnnouncement` ya tenía la forma correcta).

---

## Incremento J — Migración destructiva `063d`

```bash
alembic upgrade head        # aplica 063d: borra promotions.type/value/min_qty (+CHECKs);
                             # borra promotion_variants.promotion_id (+FK+índice+UNIQUE vieja);
                             # promotion_rule_id -> NOT NULL; UNIQUE nueva
                             # (promotion_rule_id, product_variant_id).
alembic downgrade -1        # recrea promotions.type/value/min_qty y promotion_variants.promotion_id
                             # (nullable); repuebla desde promotion_rules — ver limitación de
                             # data-model.md §"downgrade()" si alguna promoción ya tiene > 1 regla.
alembic upgrade head
```

**Verificación J**:

1. `promotions.type`/`value`/`min_qty` y `promotion_variants.promotion_id` ya no existen. Crear una
   fila directamente contra `promotion_rules` sin pasar por el CRUD sigue exigiendo sus `CHECK`s
   (`ck_promotion_rule_*`).
2. Con una promoción que llegó a tener 2 reglas (creada después del Incremento H), el `downgrade`
   de `063d` recrea `promotions.type/value/min_qty` **vacío/nullable** para esa fila (no puede
   aplanar 2 reglas en una), con aviso en el log — mismo criterio ya aceptado para el downgrade de
   `063b`.

```bash
python -m unittest discover -s app/characterization_tests -p 'test_*.py' -v
python app/scripts/test_promotions_rules.py
```

---

## Verificación final — no regresión

**Resultado esperado**: la suite completa pasa. En particular:

- Los totales de cobro de cualquier promoción con **exactamente 1 regla** (todo lo migrado por
  `063c` que nadie tocó después) son **idénticos** a los del Paso 0 — la partición no cambia ningún
  número cuando solo hay una regla por promoción.
- Los `CONGELA` que no tocan promociones (`test_orders_tables_advanced.py`,
  `test_orders_consolidation.py`, resto de `test_table_sessions_service.py`) pasan **sin editar**.
- `test_promotions_rules.py` (CI) reescrito en verde, con el montaje de sus casos pasando por
  `add_rule_to_promotion` en vez de campos sueltos en la promoción.

## Antes de dar por completada esta revisión (Principio X)

- [ ] Los Acceptance Scenarios nuevos/ajustados de las 6 historias (en particular US1 escenario 2
      y US5 escenarios 1/5, los que ejercitan explícitamente el comportamiento multi-regla),
      satisfechos y cubiertos por tests.
- [ ] FR-001a probado en tres capas: validación de cliente (frontend), 409 del servicio, y
      ausencia de invariante roto en la base de datos tras la migración `063c` (ninguna variante
      terminó en dos reglas de la misma promoción).
- [ ] `063c` y `063d` (`upgrade`/`downgrade`) probadas contra una base real; ninguna
      `Sale`/`Invoice` cambió de importe; la verificación de regresión 1:1 del Paso 1 pasó antes de
      tocar el motor.
- [ ] `spec.md` §"Cambios de comportamiento" ítem 9 (introducción de `Regla`) no requiere entrada
      nueva en `registro-de-anomalias.md` — no es un cambio de comportamiento de producción
      (ninguna de las dos ramas está en `main`); confirmar que sigue siendo así al momento de
      mergear (si `main` avanzó y una de las dos ramas ya se mergeó entretanto, reevaluar).
- [ ] El límite de reglas por plan de suscripción **no** se implementa aquí — sigue fuera de
      alcance (`spec.md` §Out of Scope); si se decide implementarlo, es una extensión de la spec
      033 sobre el recurso "reglas activas".
