# Quickstart: validar la refactorización de promociones (modelo por conjunto de variantes)

Guía de ejecución para comprobar que la implementación cumple `spec.md`. No repite firmas ni
columnas ya detalladas en [data-model.md](./data-model.md) y [contracts/](./contracts/) — solo
enlaza a ellas.

**Prerequisitos**: venv de `pos-backend`, ejecutado desde la raíz de `../pos-backend` (sibling de
este repo). Frontend: `../pos-heladeria` con su toolchain habitual.

```bash
cd ../pos-backend
source env/bin/activate
```

Los characterization tests usan SQLite en memoria (sin PostgreSQL, sin ejecutar la migración
`@for_each_tenant_schema`). La migración y su **paso de datos** se prueban contra una base real
(`alembic upgrade head` / `alembic downgrade -1`).

---

## Paso 0 — Línea base antes de tocar código

```bash
python -m unittest app.characterization_tests.test_promotions_router -v
python -m unittest app.characterization_tests.test_menu_router -v
python -m unittest app.characterization_tests.test_orders_checkout -v
python -m unittest app.characterization_tests.test_cart_service -v
python -m unittest app.characterization_tests.test_table_sessions_service -v
python -m unittest app.characterization_tests.test_orders_tables_advanced -v
python -m unittest app.characterization_tests.test_orders_consolidation -v
python -m unittest app.characterization_tests.test_promotions_presentation_pricing -v   # spec 040, se eliminará
python app/scripts/test_promotions_rules.py                                             # CI, se reescribirá
```

**Resultado esperado**: todo en verde. Se fija la línea base para volver a correr al final; los
tests que no tocan promociones deben terminar idénticos, y los que sí, con reescrituras que
**citan A-58…A-65** ([contracts/migracion.md](./contracts/migracion.md) §2).

---

## Prerrequisito de Principio II — `registro-de-anomalias.md`

Antes de tocar código: las 8 entradas **A-58 … A-65** están en
[`specs/000-reconocimiento/registro-de-anomalias.md`](../000-reconocimiento/registro-de-anomalias.md)
(detalle en [research.md](./research.md) D18), y la entrada **A-29** quedó marcada como resuelta
por esta spec. Sin esto, ningún commit del incremento A puede citar la decisión que lo autoriza
(Principio III, FR-028).

---

## Paso 1 — Migración, contra PostgreSQL real

Sembrar un tenant con promociones de los cinco tipos (`percent` de producto, `percent` de
categoría, `percent` global, `fixed`, `combo` de 2 componentes, `qty_price`,
`qty_price_presentation`) y al menos una `Sale` + `Invoice` con descuento ya emitidas.

### 1a — Revisión aditiva `063a` (Incremento A)

```bash
alembic upgrade head        # aplica 063a: crea promotion_variants; migra percent -> conjunto;
                            # finaliza combo/fixed/qty_price/qty_price_presentation (type sin cambio);
                            # agrega closed_by_refactor_at, sales/invoices.applied_promotions,
                            # customer_orders.discount / .applied_promotions, ck_promotion_min_qty;
                            # ck_promotion_type AMPLIADO con 'package_price'.
                            # NO borra ninguna tabla vieja. La suite existente sigue en verde.
alembic downgrade -1        # revierte 063a sin error (tablas viejas nunca se tocaron)
alembic upgrade head
```

**Verificación 1a** (contra PostgreSQL **real**):

1. Cada `percent` tiene ahora filas en `promotion_variants` = las variantes **activas** de su
   alcance previo (foto fija). `type`, `value`, `status`, vigencia sin cambio. (US6-CA1, FR-026)
2. Cada `combo` / `fixed` / `qty_price` / `qty_price_presentation` no terminal tiene
   `status = 'finished'`, `closed_by_refactor_at` no nulo y **`type` sin cambio**.
   `GET /promotions?closed_by_refactor=true` las lista. (US6-CA2/CA3/CA4, FR-025)
3. `promotion_targets` / `presentations` / `promotions.priority` / `product_variants.presentation_id`
   **siguen existiendo** — el motor viejo aún corre.
4. La `Sale` y la `Invoice` previas: `discount`, `total`, `number` **sin cambio**;
   `applied_promotions = '[]'`. (US6-CA5, Principio VII)

### 1b — Revisión destructiva `063b` (Incremento F, tras la Phase 8)

```bash
alembic upgrade head        # aplica 063b: borra promotion_targets / _combo_items /
                            # _presentation_rules / presentations / product_variants.presentation_id
                            # / promotions.priority; drop ck_promotion_qty_price_pack;
                            # ck_promotion_type -> "IN ('percent','package_price') OR status='finished'"
alembic downgrade -1        # recrea la estructura de spec 013/040 vacía, sin error
alembic upgrade head
```

**Verificación 1b**:

5. `presentations`, `promotion_targets`, `promotion_combo_items`, `promotion_presentation_rules`
   ya no existen; `promotions.priority` y `product_variants.presentation_id` ya no existen.
   Crear una promoción **viva** de tipo `combo`/`qty_price`/`fixed` es rechazado por
   `ck_promotion_type`; las `finished` migradas siguen legibles. (FR-027, FR-002, FR-003, A-63)
6. Tras `downgrade` de `063b`: las tablas de spec 013/040 vuelven a existir vacías, `priority` y
   `presentation_id` vuelven; **ninguna** `Sale`/`Invoice` cambió de importe
   ([data-model.md](./data-model.md) §Rollback).
7. **SQLite no ejecuta ninguno de estos pasos** — la creación de una promoción `package_price`
   (13 chars) y el `INSERT` en `promotion_variants` contra Postgres real cierran lo que el char
   test no ve.

---

## US1 — El administrador arma una promoción sobre un conjunto explícito de variantes

Archivo: `test_promotions_rules_admin.py` (nuevo).

1. `POST /promotions` `type=package_price`, `value=12000`, `min_qty=2`, `variant_ids=[8 variantes
   "Pequeño con licor"]` → 201; `status='draft'`; `condition_text` = "Llevando 2 de estas 8
   variantes pagas $12.000"; `variants[]` con `unit_price` de cada una. (CA1, CA5)
2. `POST` con `variant_ids=[]` → **422** "Una promoción necesita al menos una variante". (CA3)
3. `POST` `type=percent`, `value=150` → **422**. (CA4)
4. El frontend usa el filtro "categoría = Granizados con licor" para poblar el selector, guarda,
   y una variante creada **después** en esa categoría **no** aparece en `variants[]` de la promo.
   (CA2, FR-003/FR-004)
5. `POST` `type=package_price`, `value=12000`, `min_qty=2`, conjunto = {Pequeño con licor $8.000,
   Pequeño **sin** licor $6.000} → **409** FR-016 (2×$6.000 = $12.000, no descuenta). (SC-002)

```bash
python -m unittest app.characterization_tests.test_promotions_rules_admin -v
```

---

## US2 — El cajero cobra y el paquete combina variantes distintas del conjunto

Archivo: `test_promotions_service.py` (nuevo). Precios del catálogo real (Assumptions).

| # | Promoción activa | Pedido | Total esperado | FR/CA |
|---|---|---|---|---|
| 1 | "2X Pequeños con licor $12.000" (8 variantes, $8.000 c/u) | 1 Ojo de Diablo Pequeño + 1 Perla Negra Pequeño | **$12.000** + etiqueta | CA1, SC-008 |
| 2 | ídem | 3 unidades de 3 variantes distintas | **$20.000** (1 grupo $12.000 + 1 suelta $8.000); la suelta = variante de id más alto | CA2 |
| 3 | ídem | pedido #2 en otro orden de captura | **$20.000**, mismo reparto | CA3, SC-005 |
| 4 | "10% en granizados" (`min_qty` 1) | 1 Grande con licor $15.000 + 1 Mediano sin licor $8.000 | **$20.700** ($23.000 − $1.500 − $800) | CA4 |
| 5 | "15% llevando 3 medianos" (`min_qty` 3, conjunto = Mediano con y sin licor) | 2 Mediano con licor $11.000 + 2 Mediano sin licor $8.000 | **$33.500** (grupo de las 3 más caras $30.000, −15% = −$4.500; 4ª a $8.000); reparto −$3.300 / −$1.200 | CA5, SC-005 |
| 6 | cualquiera con `min_qty` no alcanzado | — | sin descuento ni etiqueta | CA6 |
| 7 | `daysOfWeek={martes}` | cobrado miércoles | sin descuento | CA7 |
| 8 | ventana 15:00–17:00 | 14:59 vs 15:01 | 14:59 sin, 15:01 con | CA8 |
| 9 | "2X Pequeños con licor $12.000" | 2 unidades, una variante desactivada tras agregarse | **$16.000**, sin etiqueta (0 grupos) | CA9, FR-011 |
| 10 | "3 Pequeños sin licor por $16.000" (`min_qty` 3, $6.000 c/u) | 3 unidades de 3 variantes | **$16.000** exacto; residuo $1 a la variante de id más alto ($5.334 vs $5.333); Σ descuentos por línea = $2.000 al peso, en cualquier orden | SC-005 (división no exacta) |
| 11 | "2X Pequeños con licor $12.000" | 1 Ojo de Diablo $8.000 + 1 Manzana Verde sin licor $6.000 + 1 Perla Negra $8.000 (conjunto mixto, `min_qty` 2) | **$18.000** (grupo toma las 2 de $8.000 → $12.000; la de $6.000 suelta) | Assumptions |

Coexistencia con la terminal (US2-CA10): con "2X" activa y 1 unidad elegible en el pedido, el
preview de la terminal muestra la condición pero **sin** descuento efectivo; al agregar la 2ª,
muestra −$4.000 (FR-023) — calculado por el preview del cobro, no localmente.

```bash
python -m unittest app.characterization_tests.test_promotions_service -v
python -m unittest app.characterization_tests.test_orders_checkout -v
```

---

## US3 — Vigencia por días/franja; sin solape real

Archivo: `test_promotions_rules_admin.py`.

1. Activar "10% en granizados" todos los días; activar "20% en granizados" (variantes que se
   solapan) todos los días → **409** nombrando la promoción y las variantes compartidas. (CA2)
2. Cambiar la 2ª a `daysOfWeek={martes}` + ventana 00:00–14:59 mientras la 1ª va de 15:00 a
   cierre → **permitido** (ventanas no se cruzan). (CA3)
3. "Happy hour 22:00–01:00, viernes y sábado, 25% en litros" + 1 Granizado Litro con licor
   $28.000 cobrado **sábado 00:30** → descuento **sí** aplica: la madrugada del sábado pertenece
   al viernes (A-57 conservado). Total **$21.000**. (CA1)
4. Promo `Activa` sin franja horaria que incluye "Granizado Mediano con licor"; intentar activar
   otra sobre esa variante con franja 15:00–17:00 y días/fechas que se cruzan → **409**: la promo
   sin franja cubre todas las horas (FR-014a). (CA5)
5. Guardar ventana 22:00–02:00 → aceptada (cruza medianoche). (CA4)

```bash
python -m unittest app.characterization_tests.test_promotions_rules_admin -v
```

---

## US4 — El cliente del menú QR ve la promoción y su precio efectivo

Archivo: `test_menu_router.py` (clase reescrita; los `test_a08_*` re-congelan la vigencia en hora
local con conjunto de variantes, sin tocar `_build_menu`).

1. "2X Pequeños con licor $12.000" **vigente en este instante** → `GET /menu/promotions` (y la
   clave `promotions` del flujo QR con token) incluye el anuncio "Llevando 2 de estos 8 sabores
   pagas $12.000", visible con carrito vacío. (CA1)
2. Misma promo `active` pero **fuera** de su ventana de día u hora en ese momento → la lista no la
   incluye. (CA2, SC-007)
3. Carrito con 1 unidad elegible de una promo `min_qty` 3 → `discounted_total` = precio normal +
   condición visible; al llegar a 3 → `discounted_total` refleja el precio de paquete. (CA3)
4. Regresión: `_build_menu(db)` sigue devolviendo `list[MenuCategoryResponse]`; `test_a08_*`
   pasan.

```bash
python -m unittest app.characterization_tests.test_menu_router -v
```

Verificación manual del frontend: abrir el menú público por QR dentro y fuera de la ventana y
confirmar que el banner aparece/desaparece.

---

## US5 — Duplicar, editar una promoción activa, cambiar de estado

Archivo: `test_promotions_rules_admin.py`.

1. Promo `Activa`: editar `name`, `description`, `ends_at`, `days_of_week`, `start_time`/`end_time`
   → 200. (CA1)
2. Misma promo: `PATCH` con `value` distinto o `PATCH /shape` con `variant_ids` distinto → **422**
   / **409** que sugiere duplicar. (CA2)
3. Promo `Finalizada`: `PATCH /status {"status":"active"}` → **409** (terminal). (CA3)
4. `POST /promotions/{id}/duplicate` → copia `Borrador` con mismo tipo/valor/`min_qty`/conjunto,
   nombre distinto. (CA4)
5. Usuario cajero: `POST /promotions` → **403**. (CA5)

```bash
python -m unittest app.characterization_tests.test_promotions_rules_admin -v
```

---

## US6 — Migración de las promociones existentes

Archivo: `test_promotions_migration.py` (nuevo; ejercita el callable de migración de datos sobre
SQLite con el esquema pre-refactor sembrado, más la verificación end-to-end del Paso 1 contra
Postgres).

1. `percent` del 10% sobre categoría "Granizados" → tipo porcentaje, valor 10, conjunto = todas
   las variantes activas de la categoría al migrar; estado y vigencia conservados. (CA1)
2. `combo` "1 Litro + 1 Cono por $30.000" → `Finalizada`, `closed_by_refactor_at` no nulo,
   aparece en `GET /promotions?closed_by_refactor=true`; líneas de venta históricas sin cambio.
   (CA2)
3. `qty_price_presentation` activa → `Finalizada` + aviso. (CA3)
4. `fixed` "$2.000 de descuento por línea" → `Finalizada`; **no** se convierte en porcentaje ni
   en precio de paquete. (CA4)
5. `Sale` ya emitida con descuento de cualquier tipo → `discount`, `total`, factura sin cambio.
   (CA5)

```bash
python -m unittest app.characterization_tests.test_promotions_migration -v
```

---

## Verificación final — no regresión

```bash
python -m unittest discover -s app/characterization_tests -v
python app/scripts/test_promotions_rules.py
```

**Resultado esperado**: la suite completa pasa. En particular:
- `test_promotions_rules.py` (CI) reescrito: consumo codicioso, grupos completos, reparto al peso
  con residuo (SC-005), tope al precio normal, vigencia local + cruce de medianoche + **A-57
  intacto**, `type` de **entrada** ∈ {percent, package_price} (las promociones `finished` migradas
  conservan su `type` histórico).
- Los `CONGELA` de `test_orders_checkout.py` / `test_table_sessions_service.py` /
  `test_orders_tables_advanced.py` / `test_cart_service.py` que **no** tocan promociones pasan
  **sin editar**.
- Sin ninguna promoción `active`, todos los totales de cobro coinciden con la línea base del
  Paso 0.
- Los ficheros de la spec 040 (`test_promotions_presentation_*`, `test_presentations_service.py`,
  `presentation_fixtures.py`) ya no existen.

## Frontend

```bash
cd ../pos-heladeria
# runner de tests habitual del repo
ng build      # el módulo presentations y scope-picker deben haber salido sin romper el build
```

- `modules/promotions/` — formulario de **dos tipos** (Descuento % / Precio de paquete), selector
  de conjunto de variantes con filtros por producto/categoría/texto (solo para poblar), resumen
  legible antes de guardar (FR-005), diálogos de FR-014 / FR-016 / FR-018, banner de FR-025.
- `modules/presentations/` — **eliminado**; `/dashboard/presentations` fuera de la navegación.
- `modules/promotions/components/scope-picker.component.ts` — **eliminado**.
- `modules/products/pages/product-form.component.ts` — sin selector de presentación;
  `modules/products/services/product.service.ts` — sin `presentation_id` en payloads/mapeos.
- `modules/tables/services/diner.service.ts` — `MenuPromotionAnnouncement.rules` con la forma
  nueva (`{ text, variant_count, min_qty, value }`), sin `presentation_name` / `pack_price`.
- `modules/tables/components/combo-select.component.ts` — **eliminado**; sin flujo "agregar combo"
  en la terminal.
- `modules/tables/pages/public-menu.component.ts` — banner de anuncio por conjunto de variantes.
- `modules/tables/services/pos-terminal.store.ts` — condición siempre visible + descuento
  efectivo al alcanzar `min_qty` (del preview del cobro).

Comprobación manual (`ng serve`, admin → Promociones → "＋ Nueva promoción"):
1. El selector de tipo muestra **2 opciones** ("Descuento %", "Precio de paquete") — no combo, no
   qty_price, no presentación.
2. "Precio de paquete" → elegir 8 variantes con el filtro de categoría → el resumen muestra
   "Llevando 2 de estas 8 variantes pagas $12.000" y la lista con precios.
3. Activar una 2ª promo con variante compartida y horario que se cruza → diálogo del 409 de
   FR-014.
4. Abrir una promo `Activa` → solo nombre/descr./fin/días/horas editables.
5. Si hubo migración: el banner lista las promociones finalizadas por el refactor.

## Antes de dar la spec por completada (Principio X, Constitución §"Un spec se considera completado")

- [ ] Los Acceptance Scenarios de las 6 historias, satisfechos y cubiertos por tests.
- [ ] Los 8 SC de `spec.md` verificados (en particular SC-005 reparto al peso —caso de división
      no exacta— y SC-003 bloqueo/permiso de solape).
- [ ] `test_promotions_router.py` y los `test_a08_*` de `test_menu_router.py` en verde tras la
      reescritura que **cita A-58…A-65**; `test_promotions_rules.py` (CI) reescrito en verde.
- [ ] Ambas revisiones (`063a` aditiva y `063b` destructiva) `upgrade`/`downgrade` probadas contra
      una base real; ninguna `Sale`/`Invoice` previa cambió de importe; tras `063a`, la suite
      existente quedó en verde antes de tocar el motor.
- [ ] Las 8 entradas **A-58 … A-65** están en `registro-de-anomalias.md` y **A-29** quedó marcada
      como resuelta por esta spec.
- [ ] Los ficheros, el módulo y la integración de `Presentation` (incluida la de `api/v1/catalog/`,
      `product.service.ts` y `diner.service.ts`) eliminados; el resto de spec 040 (A-57, anuncio en
      menú QR, vigencia día/hora) conservado.
