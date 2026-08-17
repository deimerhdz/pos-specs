# Qué módulo extraer primero — análisis y mapa de consumidores del motor elegido

**Fecha**: 2026-08-17
**Alcance**: decidir, dentro de "Pedidos de Mesa por QR e Inventario", qué módulo conviene
extraer primero al aplicar el Principio III de la [Constitución](../../.specify/memory/constitution.md)
("Estrangulamiento antes que Reescritura... **prohibido reescribir más de un módulo a la vez**").
**Método**: lectura directa de código (`grep -rn` sobre `../pos-backend` y `../pos-heladeria`) y
cruce contra [`registro-de-anomalias.md`](./registro-de-anomalias.md) (49 anomalías) y el
paquete `app/characterization_tests/` ya existente en `pos-backend`. Toda afirmación cita
fichero y línea. Este documento no propone corregir nada — solo ordena la evidencia para decidir
la secuencia de extracción.

---

## 1. Candidatos evaluados

El alcance "Pedidos de Mesa por QR e Inventario" (ver
[`mapa-sistema.md`](./mapa-sistema.md) §2.2 y [`flujo-pedido-qr.md`](./flujo-pedido-qr.md))
cubre cinco agrupaciones de código con responsabilidad propia:

| Candidato | Ficheros | Qué hace |
|---|---|---|
| `cart` | `app/api/v1/cart/{router,service}.py` | Carrito del comensal (unirse, añadir/editar líneas, enviar a cocina) |
| `table_sessions` | `app/api/v1/table_sessions/{router,service}.py` | Sesión de mesa, reparto, cobro/cierre |
| `orders` | `app/api/v1/orders/{service,checkout,consolidation,consumption,kitchen,tables_advanced}.py` | Confirmación, cocina, cobro, mesas físicas |
| `inventory` | `app/api/v1/inventory/{stock,service}.py` | Kardex, compras, ajustes |
| **motor de catálogo** | `app/api/v1/catalog/{line_pricing,consumption_plan}.py` | Precio de línea + "qué se descuenta de inventario" |

`promotions`, `cash`, `sales` (mostrador) y `auth` quedan fuera: no son específicos de
"pedidos de mesa por QR e inventario" (aunque `sales` reutiliza el motor de catálogo, ver §4).

---

## 2. Criterio 1 — Valor de negocio (decisiones pendientes del registro de anomalías)

Se contó, para cada candidato, cuántas entradas de `registro-de-anomalias.md` tocan **su propio
código** (no solo lo que consume) y qué tratamiento tienen.

| Candidato | Entradas que lo tocan | Detalle |
|---|---|---|
| **motor de catálogo** (`line_pricing.py` + `consumption_plan.py`) | **9**: A-02, A-03, A-04, A-05, A-06, A-32, A-33, A-36(parcial), A-47(parcial) | 2 `[PROTEGIDA]` (invariantes ya rotos una vez, deben congelarse), 1 bug histórico con corrección de una línea ya identificada (A-04), 4 `PENDIENTE` con tratamiento "documentar sin especificar" ya cerrado en la entrevista |
| `orders` (5 ficheros) | 7: A-01(caminos B/C), A-04(el bug vive en `consolidation.py`), A-16, A-25, A-26, A-29(parcial), A-38 | repartidas entre `checkout.py`, `consolidation.py`, `kitchen.py`, `tables_advanced.py` — ningún fichero individual concentra más de 2 |
| `table_sessions` | 5: A-01(camino A, correcto), A-15, A-17(parcial), A-29(parcial), A-38(parcial) | 1 `[PROTEGIDA]` (A-15, blindaje de split) |
| `inventory` | 2: A-13, A-35(cluster de 4 sub-hallazgos) | ningún `[PROTEGIDA]`; ya es el patrón de referencia que otros deberían copiar (`SELECT...FOR UPDATE`, citado en A-16/A-17) |
| `cart` | 2: A-08(reloj UTC), A-17(parcial, R16) | bajo impacto individual |

El motor de catálogo concentra casi el doble de anomalías que su competidor más cercano
(`orders`), y las tiene **en solo 446 líneas repartidas en 2 ficheros** (`line_pricing.py`:
220, `consumption_plan.py`: 226) en vez de dispersas en 5 ficheros de ~2.000 líneas
combinadas. Es además el único candidato con dos invariantes `[PROTEGIDA]` — comportamiento que
ya se rompió una vez en producción (A-02: doble descuento de 140g) y que la Constitución exige
fijar como caso de test explícito antes de tocar nada alrededor.

`A-04` es el hallazgo más fuerte de todo el registro para justificar "extraer esto primero": es
el único con prueba directa de `git log`/`git show` de cómo se rompió (regresión de merge entre
`03469ca` y `ee94f30`), tiene corrección de una sola línea ya identificada
(`orders/consolidation.py:199`, falta `variant=variant`), y sigue activo hoy en el camino real
que usa el mesero.

---

## 3. Criterio 2 — Cobertura actual de characterization tests

`pos-backend` no tiene una carpeta `tests/` clásica; su equivalente es
`app/characterization_tests/` (ver `__init__.py:1-39`, que cita explícitamente el Principio II
de la Constitución: "el comportamiento actual es sagrado"). Se contaron los métodos `def test_`
de cada fichero:

| Fichero de test | Métodos | Módulo que congela |
|---|---|---|
| `test_catalog_line_pricing.py` | **25** | `catalog/line_pricing.py` |
| `test_catalog_consumption_plan.py` | **16** | `catalog/consumption_plan.py` |
| `test_golden_master_pricing_consumption.py` | 4 (+ 8-12 casos de negocio encadenados vía `golden_master_core.py`) | **ambos ficheros del motor, encadenados como en producción** |
| `test_inventory_stock.py` | 16 | `inventory/stock.py` |
| `test_catalog_service_sku.py` | 12 | `catalog/service.py` (SKU, no el motor) |
| `test_invoices_full_number.py` | 6 | `invoices/schemas.py` |
| `test_core_inventory_reasons.py` | 5 | `core/inventory_reasons.py` |
| `test_core_units.py` | 5 | `core/units.py` (código muerto, A-31) |
| — | **0** | `orders/*`, `table_sessions/*`, `cart/*`, `sales/*`, `promotions/*`, `cash/*` |

**El motor de catálogo ya es, con diferencia, el subsistema mejor cubierto de todo
`pos-backend`**: 45 tests unitarios propios más un golden master dedicado
(`golden_master_core.py:1-24` lo describe literalmente como "el flujo central de negocio...
identificado en `flujo-pedido-qr.md`"). `orders`, `table_sessions`, `cart`, `sales`, `promotions`
y `cash` — es decir, el resto del alcance "pedidos de mesa por QR" — no tienen ni un solo
characterization test todavía.

Esto invierte la lectura ingenua de "extraer lo que ya está probado es poco interesante": bajo
el Principio II de la Constitución (los characterization tests son el árbitro), **ningún módulo
puede extraerse con seguridad hasta tener esa red**. El motor de catálogo es el único candidato
que ya la tiene completa, incluido el golden master que verifica el encadenamiento real
(`add_item` → validar opciones → fijar precio → calcular consumo) tal como lo ejecutan los tres
caminos de creación de pedido. Extraerlo primero es extraer el único módulo que ya cumple el
requisito previo que la propia Constitución exige para los demás.

Nota: 12 scripts legados en `app/scripts/test_*.py` (no characterization tests formales, sin
`unittest.TestCase`, no ejecutados en CI — A-27) dan cobertura adicional indirecta a
`table_sessions` (`test_table_sessions.py`, `test_table_release.py`, `test_split_blindaje.py`),
`orders`/inventario (`test_cancel_inventory.py`, `test_receta_obligatoria.py`) y sesiones
(`test_session_ttl.py`). No se cuentan junto a los characterization tests porque no siguen su
convención (`"CONGELA comportamiento actual:"`, Principio II) ni corren automáticamente.

---

## 4. Criterio 3 — Acoplamiento

### 4.1 Quién importa el motor (consumidores de producción)

Búsqueda exhaustiva (`grep -rn` sobre `pos-backend`, excluyendo `env/` y `__pycache__`) de todo
import de `app.api.v1.catalog.line_pricing` o `app.api.v1.catalog.consumption_plan`:

| Función | Consumidor (fichero:línea de import) | Línea(s) de llamada | Pureza |
|---|---|---|---|
| `compute_line_price` | `sales/service.py:23` | `:75` | **Pura** — `Decimal` de variante + opciones, sin `db`/red/reloj |
| | `orders/consolidation.py:28` | `:200` | |
| | `orders/service.py:28` | `:117` | |
| | `orders/kitchen.py:22` | `:149` | |
| | `cart/service.py:31-36` | `:299`, `:380` | |
| `load_valid_options` | `sales/service.py:23` | `:74` | Impura — DB (`get_or_404`, `Option`); lanza 404/422 |
| | `orders/consolidation.py:28` | `:199` ⚠️ **sin `variant=`, es A-04** | |
| | `orders/service.py:28` | `:102` (sí pasa `variant=variant`) | |
| | `orders/kitchen.py:22` | `:125` | |
| | `cart/service.py:31-36` | `:286`, `:363` (ambos con `variant=variant`) | |
| `check_availability` | `cart/service.py:31-36` | `:292`, `:329`, `:486` | Impura — DB (`db.get(InventoryItem,...)`); lanza 409. **Único consumidor: solo `cart`** |
| `required_consumption` | `cart/service.py:31-36` (re-exportado vía `line_pricing.py:31-36`) | `:197`, `:290`, `:325`, `:374` | Impura — envuelve `plan_line_consumption` (DB) |
| | `sales/consumption.py:17-21` (import directo de `consumption_plan`) | `:58` | |
| | `orders/consumption.py:19-23` (import directo de `consumption_plan`) | `:47` | |
| `plan_line_consumption` | `sales/consumption.py:17-21` | `:63` | Impura — llama internamente `load_recipe`/`load_variant_groups` (DB); el resto es aritmética pura |
| | `orders/consumption.py:19-23` | `:95`, `:120` | |
| `ensure_lines_consume_inventory` | `sales/consumption.py:17-21` | `:49` | Impura — la más costosa: internamente dispara `required_consumption`→`plan_line_consumption`→`load_recipe`+`load_variant_groups`, más `variant_label` y `group_discounts`→`load_variant_groups` de nuevo por cada línea rechazada |
| | `orders/consumption.py:19-23` | `:32` | |

**Funciones sin consumidor externo** (solo se llaman entre sí dentro de los dos ficheros del
motor, más los tests): `validate_option_selection` (llamada únicamente desde
`load_valid_options:65`), `grupos_que_descuentan` (desde `validate_option_selection:133`),
`_exige_maximo` (helper privado, **pura**, desde `validate_option_selection:146,158`),
`load_recipe`, `load_variant_groups`, `group_discounts`, `variant_label` (todas internas a
`consumption_plan.py`, impuras por `db.execute`/`db.get`).

**Detalle de acoplamiento notable**: `cart/service.py` no importa `required_consumption` desde
`consumption_plan.py` directamente, sino desde el *reexport* que hace `line_pricing.py:31-36`
("`# noqa: F401 (reexport)`", comentario propio del fichero) para "no romper los imports
existentes". Cualquier extracción del motor debe decidir explícitamente si conserva ese shim o
actualiza `cart/service.py:31-36` al import directo.

### 4.2 Resumen de pureza

De las 6 funciones con consumidor de producción, **solo `compute_line_price` es pura** (sin
`db`, sin red, sin reloj). Las otras 5 reciben `db: Session` y ejecutan sus propias consultas
internamente — el motor hoy **no** separa cálculo de carga de datos; cada función pública
mezcla ambas cosas. Esto es, en sí mismo, un hallazgo relevante para la extracción: convertir el
motor en un paquete independiente exige primero separar los `load_*` (adaptadores de
persistencia) de la aritmética/validación pura que ya cubren los characterization tests — el
propio golden master (`golden_master_core.py:1-24`) ya aísla ese "núcleo de cálculo" como
concepto, aunque el código de producción todavía no lo hace.

A diferencia del motor de promociones (`promotions/service.py`, ver A-07/A-08/A-09), **ninguna
de las dos funciones del motor de catálogo depende del reloj del sistema** — su
determinismo depende solo de los datos ya cargados, lo que hace su golden master
(`pricing_consumption.master.json`) fiable byte a byte sin el riesgo de fuga de zona horaria que
sí afecta a promociones.

### 4.3 Fan-out (de qué depende el motor)

`line_pricing.py` y `consumption_plan.py` no importan ningún otro módulo de negocio de
`app/api/v1/*` — solo modelos ORM (`Option`, `OptionGroup`, `ProductVariant`, `InventoryItem`,
`Product`, `RecipeItem`, `VariantOptionGroup`) y utilidades de `app/core` (`config`, `crud`). Es
un módulo **hoja**: se puede extraer sin arrastrar cambios de lógica en `cart`, `orders` ni
`sales` — solo sus puntos de import (la tabla de §4.1), un conjunto pequeño y ya enumerado por
completo.

### 4.4 Ningún puerto duplicado en el frontend

`grep -rln` sobre `pos-heladeria/src/` no encuentra ninguna reimplementación del motor de
catálogo (a diferencia de `promotion-pricing.util.ts`, que sí porta el motor de promociones —
A-09/A-10). El único hit es un comentario documental en
`product-form.component.ts:555` ("La cantidad replica `plan_line_consumption` del backend").
Extraer este motor no exige coordinar dos implementaciones divergentes en dos repos.

---

## 5. Recomendación

**Extraer primero el motor de catálogo** (`app/api/v1/catalog/line_pricing.py` +
`consumption_plan.py`) como su propio módulo, antes que `cart`, `table_sessions`, `orders` o
`inventory`. Razones, en orden de peso:

1. **Es el único candidato con la red de seguridad que exige el Principio II ya construida**:
   45 characterization tests + golden master, contra cero en `cart`/`table_sessions`/`orders`.
   Bajo "estrangulamiento antes que reescritura" (Principio III), extraer sin esa red en los
   demás candidatos violaría el propio método que la Constitución define — este es el único
   módulo que hoy lo cumple.
2. **Concentra la mayor densidad de decisiones de negocio del registro de anomalías** (9 IDs en
   446 líneas, incluidas las dos únicas invariantes `[PROTEGIDA]` de todo el alcance QR fuera de
   seguridad/facturación, y el único bug con causa raíz confirmada por `git log`, A-04).
3. **Es un módulo hoja**: cero dependencias hacia otros módulos de negocio, fan-in acotado y
   enumerado por completo (7 ficheros consumidores, 6 funciones públicas realmente usadas). Su
   extracción no obliga a tocar la lógica de `cart`/`orders`/`sales`, solo sus imports.
   `orders` y `table_sessions`, en cambio, están mutuamente entrelazados (`table_sessions`
   depende de `orders.checkout`; `orders` depende de `cash`/`sales` vía `build_sale`) — extraer
   cualquiera de los dos primero arrastra al otro.
4. **Sin dependencia del reloj** (a diferencia de `promotions`) y **sin puerto duplicado en el
   frontend** (a diferencia de `promotions`) — dos fuentes de no-determinismo/divergencia que
   otros motores candidatos sí tienen, ausentes aquí.

**Condición para proceder**: antes de extraer, separar dentro del propio motor los `load_*`
(I/O sobre `db: Session`) de las funciones de cálculo puro (§4.2) — hoy mezcladas — porque un
paquete extraído no debería depender de una `Session` de SQLAlchemy para su lógica central. Esa
separación es un cambio de estructura, no de comportamiento, así que no requiere una nueva
decisión de negocio (Principio I) — pero sí debe mantener en verde, sin modificar, los 45
characterization tests y el golden master existentes (Principio II), y de paso corregir A-04
(`orders/consolidation.py:199`) citando esta decisión en el commit correspondiente, tal como ya
lo anticipa el "Tratamiento acordado" de A-04 en `registro-de-anomalias.md`.

**Segundo candidato, para cuando el motor esté extraído**: `orders/consumption.py` +
`sales/consumption.py` (los adaptadores que ya envuelven al motor) son el siguiente eslabón
natural — ya tienen tests propios indirectos (golden master) y su extracción sería el paso que
efectivamente conecta el motor extraído con `inventory/stock.py` (que ya tiene su propia red de
16 tests).

---

## 6. Mapa de consumidores del motor (artefacto de referencia)

```
app/api/v1/catalog/line_pricing.py (220 líneas)          app/api/v1/catalog/consumption_plan.py (226 líneas)
├─ load_valid_options(db, ids, *, variant=None)          ├─ load_recipe(db, variant_id)              [solo interno]
│  └─ llama validate_option_selection si variant         ├─ load_variant_groups(db, variant_id)      [solo interno]
├─ grupos_que_descuentan(db, links)     [solo interno]    ├─ group_discounts(db, link)                 [solo interno]
├─ _exige_maximo(gid, lo, consumen)     [privada, pura]   ├─ plan_line_consumption(db, vid, qty, opts)
├─ validate_option_selection(db, variant, options)        │  └─ usa load_recipe + load_variant_groups internamente
│  └─ llamada solo desde load_valid_options:65            ├─ required_consumption(db, vid, qty, opts)
├─ compute_line_price(variant, options)  ← ÚNICA PURA     │  └─ agrega plan_line_consumption por insumo
└─ check_availability(db, required, ...)                  ├─ variant_label(db, variant_id)             [solo interno]
   └─ único consumidor externo: cart                       └─ ensure_lines_consume_inventory(db, entries)
                                                                └─ usa required_consumption + variant_label + group_discounts
   (reexporta de consumption_plan: ConsumptionLine,
    load_variant_groups, plan_line_consumption,
    required_consumption — línea 31-36, "para no
    romper los imports existentes")

CONSUMIDORES DE PRODUCCIÓN (7 ficheros, ninguno más)
├─ app/api/v1/sales/service.py:23           → compute_line_price, load_valid_options
├─ app/api/v1/sales/consumption.py:17-21    → ensure_lines_consume_inventory, plan_line_consumption, required_consumption
├─ app/api/v1/orders/service.py:28          → compute_line_price, load_valid_options (correcto, pasa variant=)
├─ app/api/v1/orders/consolidation.py:28    → compute_line_price, load_valid_options (⚠️ A-04: sin variant=, línea 199)
├─ app/api/v1/orders/kitchen.py:22          → compute_line_price, load_valid_options
├─ app/api/v1/orders/consumption.py:19-23   → ensure_lines_consume_inventory, plan_line_consumption, required_consumption
└─ app/api/v1/cart/service.py:31-36         → check_availability, compute_line_price, load_valid_options, required_consumption
                                                (único consumidor de check_availability)

CONSUMIDORES DE TEST (app/characterization_tests/, 45 tests + golden master)
├─ test_catalog_line_pricing.py        (25 tests) → check_availability, compute_line_price, load_valid_options, validate_option_selection
├─ test_catalog_consumption_plan.py    (16 tests) → ensure_lines_consume_inventory, group_discounts, plan_line_consumption,
│                                                     required_consumption, grupos_que_descuentan (alias de line_pricing)
├─ golden_master_core.py + test_golden_master_pricing_consumption.py (4 tests, 8-12 casos encadenados)
│                                                  → check_availability, compute_line_price, validate_option_selection,
│                                                     ensure_lines_consume_inventory, plan_line_consumption
└─ app/scripts/test_variant_option_groups.py       → validate_option_selection (líneas 313,323,328,345,357;
   (legado, no en CI — A-27)                          script suelto, no característico formal)

FAN-OUT (de qué depende el motor): solo modelos ORM (Option, OptionGroup, ProductVariant,
InventoryItem, Product, RecipeItem, VariantOptionGroup) y app.core.{config,crud}.
Cero dependencias hacia otros módulos de negocio (cart/orders/sales/table_sessions/promotions).

PUERTO EN FRONTEND: ninguno (a diferencia de promotions). Único hit en pos-heladeria/src/ es un
comentario documental en product-form.component.ts:555.
```
