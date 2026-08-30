# Quickstart: validar Promociones de Precio por Cantidad por Presentación

Guía de ejecución para comprobar que la implementación cumple `spec.md`. No repite firmas ni
columnas ya detalladas en [data-model.md](./data-model.md) y [contracts/](./contracts/) — solo
enlaza a ellas.

**Prerequisitos**: venv de `pos-backend`, ejecutado desde la raíz de `../pos-backend` (sibling de
este repo). Frontend: `../pos-heladeria` con su toolchain habitual.

```bash
cd ../pos-backend
source env/bin/activate
```

Los characterization tests usan SQLite en memoria (sin PostgreSQL). La migración nueva
(`data-model.md` §Migración) sí requiere una base real para probarse end-to-end (`alembic upgrade
head` / `alembic downgrade -1`).

---

## Paso 0 — Línea base antes de tocar código

```bash
python -m unittest app.characterization_tests.test_promotions_router -v
python -m unittest app.characterization_tests.test_menu_router -v
python -m unittest app.characterization_tests.test_orders_checkout -v
python -m unittest app.characterization_tests.test_cart_service -v
python app/scripts/test_promotions_rules.py        # el único script de promociones en CI
```

**Resultado esperado**: todo en verde. `test_menu_router.py` (`"CONGELA comportamiento corregido:"`,
A-08) y `test_promotions_router.py` (tiene `CONGELA`) **no se modifican** por esta spec: `_build_menu`
no se toca (research.md D12) y `_valid_now` solo gana un `check()` nuevo en el script de CI sin
editar los existentes (research.md D14/D18, A-55). Se corren para fijar la línea base y volver a
correrlos al final sin cambios de cuerpo.

## Paso 1 — Migración (Incremento A)

```bash
alembic upgrade head        # crea presentations, promotion_presentation_rules,
                            # product_variants.presentation_id, ensancha promotions.type
                            # a varchar(50) y amplía ck_promotions_type
alembic downgrade -1        # revierte sin error y sin pérdida de dato histórico
alembic upgrade head        # vuelve a aplicar
```

**Verificación** (contra PostgreSQL **real**, no SQLite):

1. Tras `upgrade`, toda variante existente tiene `presentation_id IS NULL` (FR-008).
2. `promotions.type` es `character varying(50)` — comprobar en `information_schema.columns`. Sin la
   ampliación, `INSERT ... type='qty_price_presentation'` (22 chars) revienta con
   `StringDataRightTruncation` → 500. **SQLite no valida el ancho de `VARCHAR`, así que este fallo
   NO aparece en los characterization tests**: hay que crear una promoción `qty_price_presentation`
   contra Postgres real (p. ej. `POST /api/v1/promotions` o un char test de integración) para
   cerrarlo.
3. Tras `downgrade`, el esquema queda idéntico a antes de la spec — incluido `promotions.type` de
   vuelta a `varchar(20)` (`data-model.md` §Rollback).

## US4 (parte) — Catálogo de presentaciones y baja bloqueada

Fichero: `test_presentations_service.py` (nuevo).

1. Crear presentación "8oz"; crear "8oz" otra vez → 409 unicidad.
2. Asignar `presentation_id` de "8oz" a variantes de dos productos distintos (Ojo de Diablo, Fresa
   Boom) → ambas quedan en el alcance de cualquier regla sobre "8oz" (FR-007) — se verifica
   consultando `applicable_variant_count == 2`.
3. Crear un producto/variante **nuevo** con `presentation_id` de "8oz" → `applicable_variant_count`
   pasa a 3 sin tocar ninguna promoción (FR-019, CA-9).
4. Con una promoción `qty_price_presentation` **activa** que tiene una regla sobre "8oz": intentar
   `DELETE /presentations/{8oz}` y `PATCH {active:false}` → **409** con la lista de promociones
   (FR-020, CL-2). Pausar la promoción → la baja procede; las variantes quedan con
   `presentation_id NULL`.

```bash
python -m unittest app.characterization_tests.test_presentations_service -v
```

## US1 — El administrador configura una promoción con reglas por presentación

Fichero: `test_promotions_presentation_rules.py` (nuevo).

1. Crear promoción `type=qty_price_presentation` con dos reglas ("8oz" 2×$12.000, "16oz" 2×$16.500)
   → 201; `presentation_rules` en la respuesta con `applicable_variant_count` por regla (FR-005,
   CA-1).
2. Agregar una tercera regla repitiendo "8oz" → 422 "No puede haber dos reglas para la misma
   presentación" (FR-006 1ª parte, CA-2).
3. Con una promoción activa con regla sobre "8oz", crear/activar **otra** `qty_price_presentation`
   con regla sobre "8oz" → **409** nombrando la promoción en conflicto (FR-006 2ª parte, CA-3,
   CL-4). Repetir el intento en `PATCH /status {"status":"active"}` de una promoción creada en
   `draft` sin conflicto → mismo 409 (research.md D8).
4. Crear con ventana horaria `22:00`–`02:00` → aceptada como ventana que cruza medianoche (FR-003,
   CA-4).

```bash
python -m unittest app.characterization_tests.test_promotions_presentation_rules -v
```

## US2 — El cajero cobra pedidos que combinan productos de una misma presentación

Fichero: `test_promotions_presentation_pricing.py` (nuevo). Regla activa "2 × 8oz por $12.000",
variantes de 8oz a $7.000 c/u salvo donde se indique.

| # | Pedido | Total esperado | Qué verifica | FR/CA |
|---|---|---|---|---|
| 1 | 1× Ojo de Diablo 8oz + 1× Fresa Boom 8oz | $12.000 | paquete con productos distintos; etiqueta de promoción presente | E2, CA-3 |
| 2 | 1× Ojo de Diablo + 1× Fresa Boom + 1× Maracumango (8oz) | $19.000 | dos líneas a $6.000, una a $7.000; la suelta se decide por identificador de variante más alto | E3, CA-4 |
| 3 | pedido #2 con las líneas en otro orden | $19.000, mismo reparto | determinismo por orden | CA-5, SC-005 |
| 4 | 2× 8oz (sabores distintos) + 2× 16oz a $9.500, reglas "2×8oz $12.000" + "2×16oz $16.500" | $28.500 | un paquete por presentación ($12.000 + $16.500); nunca se mezclan presentaciones | FR-009 |
| 5 | 5× 8oz | $31.000 | 2 paquetes + 1 suelta | E-scenario 5 |
| 6 | 1× 8oz + 1× 16oz (ninguna alcanza el mínimo) | $16.500, sin etiqueta | presentación sin mínimo no descuenta | E4, CA-6 |
| 7 | pedido #2, día de la semana no incluido | sin descuento | vigencia por día | CA-7 |
| 8 | pedido #2, ventana `08:00`–`22:00`, a las 07:59 vs 08:01 | 07:59 sin descuento, 08:01 con descuento | vigencia por hora (zona del tenant) | CA-8 |
| 9 | 3× 8oz (sabores distintos) a $7.000, regla "3 × 8oz por $10.000" | $10.000 exacto | residuo del redondeo ($10.000÷3 = $3.333, resto $1): la línea de identificador de variante más alto paga $3.334; Σ descuentos por línea = $11.000, al peso, en cualquier orden | CL-9, SC-005 |
| 10 | 1× 8oz activa + 1× 8oz cuya variante quedó `active=false`, ambas a $7.000 | $14.000, sin etiqueta | la variante desactivada no es unidad elegible → 0 paquetes (no $12.000) | CL-1c, FR-015 |

Verificación transversal (SC-005): en los casos con paquete, `Σ (descuento por línea)` cuadra
**exacto** con el descuento total, al peso — sin importar el orden. El caso #9 (división no exacta)
es el que la ejercita de verdad: sin residuo, SC-005 se cumple trivialmente.

Coexistencia (FR-013 / FR-023), en `test_orders_checkout.py` (casos nuevos, sin tocar los `CONGELA`):
1. una línea elegible a la vez para un `qty_price` a nivel de producto y para la regla por
   presentación → recibe el descuento de **una sola** (la de menor total para esa línea), nunca la
   suma, nunca deja la línea peor que sin promoción.
2. **recálculo del pool** (research.md D6): 3× X (8oz) + 1× Y (8oz), todos a $7.000, regla
   "2 × 8oz por $12.000" + producto "3 × X por $15.000" → X se va al descuento de producto
   ($15.000); Y, que sola no completa un paquete, se cobra a $7.000 → **total $22.000** (no
   $21.000). El paquete se recalcula tras sacar X del pool.

```bash
python -m unittest app.characterization_tests.test_promotions_presentation_pricing -v
python -m unittest app.characterization_tests.test_orders_checkout -v
```

## US3 — Aviso de precio no uniforme y de "no es descuento real"

Fichero: `test_promotions_presentation_rules.py`.

1. Dos productos en "8oz" a precios distintos ($7.000 y $8.000). Crear una regla sobre "8oz" sin
   flag → **422** con el detalle (variantes, precios, `reference_unit_price` = $7.000). Reenviar
   con `confirm_precio_no_uniforme=true` → 201 (FR-017, CA-10).
2. Promoción ya activa; un producto de "8oz" cambia de precio después → **no** se revalida; la regla
   sigue como estaba (FR-018, CL-1). En el siguiente cobro, `reference_unit_price` se recalcula
   (FR-011, CL-3) — comportamiento esperado, sin aviso nuevo.
3. Crear una variante nueva en "8oz" con precio distinto mientras la promoción está activa → entra
   al alcance **sin** pasar por la verificación de uniformidad (CL-1b).
4. Regla "2 × 8oz por $14.000" con `reference_unit_price` $7.000 (`14000/2 == 7000`, no hay
   descuento) sin flag → **422** (FR-022); con `confirm_sin_descuento=true` → se guarda. En el
   cobro, esa regla nunca deja una línea peor que sin promoción (FR-023).

```bash
python -m unittest app.characterization_tests.test_promotions_presentation_rules -v
```

## US5 — El anuncio en el menú QR

Fichero: `test_menu_router.py` (`TestCase` nueva; los `CONGELA` de este fichero no se tocan — siguen
usando `_build_menu` sin cambios).

1. Promoción "2 × 8oz por $12.000" **vigente en este instante** (dentro de su ventana de día/hora)
   → `GET /menu/promotions` (y la clave `promotions` del flujo QR con token) incluye la promoción
   con el texto legible; visible con carrito vacío (CA-11).
2. Misma promoción `active` pero **fuera** de su ventana de día u hora en ese momento → la lista
   sale vacía para ella (aclaración 2026-08-26, SC-006).
3. Verificar que `now` se evalúa en zona horaria del tenant (no arrastra A-08): promoción con
   ventana que, en UTC, caería fuera pero en hora local del tenant está dentro → se anuncia.
4. Regresión: `_build_menu(db)` sigue devolviendo la lista de categorías sin cambios; los tests
   `CONGELA` (`test_a08_*`) pasan sin editar.

```bash
python -m unittest app.characterization_tests.test_menu_router -v
```

Verificación manual del frontend: abrir el menú público por QR dentro y fuera de la ventana y
confirmar que el banner aparece/desaparece.

## Verificación final — no regresión

```bash
python -m unittest discover -s app/characterization_tests -v
python app/scripts/test_promotions_rules.py
```

**Resultado esperado**: la suite completa pasa. En particular:
- `test_promotions_router.py` (tiene `CONGELA`) — en verde sin editar; los casos
  percent/fixed/qty_price/combo validan igual con el enum ampliado (research.md D14).
- `test_menu_router.py` (`CONGELA` A-08, `test_a08_*`) — en verde sin editar; `_build_menu` no se
  tocó (research.md D12).
- `test_promotions_rules.py` (CI) — en verde; `_in_time_window`/`_line_discount`/`best_line_discount`
  sin cambio; `_valid_now` cambia de cuerpo (FR-004, A-55) pero los `check()` previos pasan sin
  editar y hay un `check()` nuevo para la combinación con cruce de medianoche (research.md D18).
- Sin ninguna promoción `qty_price_presentation` activa, todos los totales de cobro son idénticos
  a los de la línea base del Paso 0 (aditividad segura, research.md D6).

## Frontend

```bash
cd ../pos-heladeria
# runner de tests habitual del repo
```

- `modules/presentations/` — CRUD funcional; el 409 de FR-020 muestra las promociones en uso.
- `modules/products/pages/product-form.component.ts` — selector de presentación por variante.
- `modules/promotions/pages/promotions-page.component.ts` — **formulario dedicado** de
  `qty_price_presentation` (bento del mockup de `spec.md` §Assumptions: tarjetas "Información
  General" + "Configuración de Reglas", paneles "Productos Aplicables" + "Resumen de la Regla" —
  FR-005). Aplica al crear y al editar borrador; las promos ya activas se ven en solo lectura.
  Diálogos de FR-017/FR-022, diálogo del 409 de FR-006.
  El selector "¿Qué quieres crear?" ya **no ofrece "Paquete"** (`qty_price`): las promociones
  `qty_price` que ya existen se listan y editan igual, pero no se pueden crear nuevas desde la UI.
- `modules/tables/pages/public-menu.component.ts` — banner de anuncio.

Comprobación manual del formulario nuevo (`ng serve`, admin → Promociones → "＋ Nueva promoción"):
1. El selector muestra **2 tarjetas** ("Descuento", "Paquete por presentación") — no "Paquete".
2. "Paquete por presentación" → formulario bento; agregar 2 reglas de presentaciones distintas →
   "Resumen de la Regla" las lista, "Productos Aplicables" muestra el conteo por regla.
3. Una 3ª regla no puede repetir presentación (el `<select>` ya no la ofrece).
4. "Revisar y crear" → "Guardar como borrador" → reabrir el borrador → mismo formulario.
5. Abrir una promoción `qty_price` existente (si hay) → se edita con normalidad.

## Antes de dar la spec por completada (Principio X, Constitución §"Un spec se considera completado")

- [ ] Los Acceptance Scenarios de las 5 historias, satisfechos y cubiertos por tests.
- [ ] Los 6 SC de `spec.md` verificados (en particular SC-005 reparto al peso —caso #9, división no
      exacta— y SC-006 ventana del anuncio).
- [ ] `test_promotions_router.py` y `test_menu_router.py` en verde sin edición; `test_promotions_rules.py`
      (CI) en verde con el `check()` nuevo de FR-004 y los previos intactos.
- [ ] Migración `upgrade`/`downgrade` probada contra una base real.
- [ ] La **única** entrada en `registro-de-anomalias.md` de esta spec es **A-55** (FR-004,
      `_valid_now`); la modalidad de descuento en sí no requiere registro (research.md D14/D18;
      `spec.md` §"Out of Scope", 5º ítem).
