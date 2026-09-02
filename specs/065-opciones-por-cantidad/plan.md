# Implementation Plan: Selección por cantidad en grupos de opciones

**Branch**: `065-opciones-por-cantidad` | **Date**: 2026-09-01 | **Spec**: [spec.md](./spec.md)

**Input**: Feature specification from `/specs/065-opciones-por-cantidad/spec.md`

## Summary

Esta spec agrega un segundo modo de selección a `OptionGroup` — "cantidad" — donde el cliente elige
libremente cuántas unidades de cada opción agregar (ej. 2 Bobombún + 1 Gomitas), en vez de solo
poder marcar/desmarcar hasta un máximo de opciones distintas ("conteo", el modo único de hoy).
El hallazgo central de la investigación previa (sesión `/speckit-clarify`) es que **el precio y el
consumo de inventario no necesitan enterarse del modo**: generalizando el motor compartido
(`compute_line_price`, `plan_line_consumption`, `validate_option_selection`) para que reciba pares
`(opción, cantidad)` en vez de una lista plana de opciones, y garantizando que el modo "conteo"
siempre resuelve `cantidad=1`, las fórmulas actuales (`extra_price × cantidad`,
`per_unit × cantidad`) se vuelven una generalización sin caso especial — cero cambio de
comportamiento para el catálogo existente. Solo la validación de forma
(`validate_option_selection`) y la capa de captura/visualización (carrito, comanda, recibo,
"Mis pedidos") necesitan un camino nuevo explícito.

Cero endpoints nuevos: 3 columnas nuevas en `option_groups`, 1 columna nueva en
`cart_item_options`/`order_item_options` cada una, 1 tipo interno nuevo (`ChosenOption`), y un
cambio de forma coordinado (`option_ids` → `options` con cantidad) en los cuatro puntos de entrada
reales que hoy aceptan una selección de opciones.

## Technical Context

**Language/Version**: Backend — Python 3.14.4 (venv `pos-backend/env`). Frontend — TypeScript
5.9.2 (Angular 21.1.x, standalone components + signals), Node 24.16.0.

**Primary Dependencies**: Ninguna dependencia nueva en ningún repositorio.
- Backend: SQLAlchemy + Alembic (ya en uso), `app.catalog_engine.core`/`pricing`/`consumption` (ya
  extraídos en spec 014, se les agrega un tipo `ChosenOption` y se generalizan tres funciones sin
  cambiar su ubicación).
- Frontend: reutiliza `MenuLookupIndex` (`modules/tables/services/menu-lookup.ts`), el mismo widget
  visual de cantidad ya usado para la cantidad de línea completa en
  `product-select.component.ts` (`dec()`/`inc()`).

**Storage**: PostgreSQL 16, schema-per-tenant. **1 migración nueva**: `option_groups` gana
`selection_mode` + dos topes nulables; `cart_item_options`/`order_item_options` ganan `quantity`
(default `1`). Sin backfill por consulta — todo dato histórico ya es "conteo, cantidad 1" por
construcción (data-model.md).

**Testing**: Backend — `unittest` vía `python -m unittest discover -s app/characterization_tests -p
'test_*.py'`, extendiendo `test_catalog_option_groups_pricing_type.py` (spec 063/064) con un
archivo hermano para "cantidad", y ajustando cualquier test que construya `CartItemIn`/
`OrderItemCreate`/línea de venta a mano con la forma vieja `option_ids`. Frontend — Vitest vía
`ng test`; `product-select.component.spec.ts` se extiende con los casos de "cantidad".

**Target Platform**: Linux server (API `pos-backend`) + navegador (SPA `pos-heladeria`).

**Project Type**: Web application (backend FastAPI + frontend Angular, repos sibling de
`pos-specs`, estructura ya establecida por specs 002-063/064).

**Performance Goals**: Ninguno nuevo. `ChosenOption` es una tupla en memoria, sin consulta
adicional; la validación de topes de "cantidad" reutiliza el mismo `load_variant_groups` ya cargado
por `validate_option_selection`, sin `SELECT` extra.

**Constraints**:
- **Cambio de contrato coordinado, no aditivo**: `option_ids: list[UUID]` se reemplaza (no se le
  agrega un campo al lado) por `options: list[OptionSelectionIn]` en `CartItemIn`/`CartItemUpdate`,
  el schema de línea de `POST /orders`, el de `orders/consolidation.py`, y el de venta de mostrador
  — los dos repos deben desplegarse juntos, mismo criterio que ya rige cualquier cambio de contrato
  en este proyecto (Constitución §Alcance).
- **No romper el modo "conteo"**: toda opción elegida en un grupo "conteo" debe resolver
  `quantity=1`; el motor de precio/consumo generalizado (`× quantity`) depende de esa garantía para
  no cambiar de comportamiento — se hace cumplir con una validación explícita (`422` si
  `quantity>1` en "conteo"), no con una suposición silenciosa.
- **Ningún dato histórico se pierde ni se recalcula**: la migración no reescribe ningún precio,
  consumo, pedido ni venta ya confirmados (Principio VII); `quantity=1` es el valor correcto para
  el 100% de las filas existentes, no una aproximación.
- **Seis superficies de renderizado a actualizar de forma idéntica** (comanda, detalle de orden,
  "Mis pedidos", panel de validación de pago, recibo impreso, carrito en vivo) — se centralizan en
  un único método nuevo (`optionLabelWithQuantity`) para no duplicar el formato "2x Nombre" seis
  veces con el riesgo de que diverjan.

**Scale/Scope**: 3 tablas backend modificadas (`option_groups` +3 columnas, `cart_item_options`
+1, `order_item_options` +1), 0 tablas nuevas. Backend: 1 migración, 1 tipo nuevo (`ChosenOption`),
4 funciones de `catalog_engine` generalizadas a `ChosenOption` (`compute_line_price`,
`validate_option_selection`, `plan_line_consumption`, y — hallazgo de `/speckit-analyze`,
ver Constitution Check — `required_consumption`/`ensure_lines_consume_inventory`, que viven en el
mismo módulo `consumption.py` y son la vía real de deducción/pre-chequeo de stock, no solo
`plan_line_consumption`), 1 función de carga extendida (`load_valid_options`), 4 schemas Pydantic
con el mismo cambio de forma (`option_ids`→`options`), y toda la cadena de consumo real de
inventario (`app/api/v1/orders/consumption.py` — 6 funciones — y
`app/api/v1/sales/consumption.py` — `_sale_item_options`/`deduct_sale`, que lee el snapshot JSONB
de `SaleItem.options`), además de 3 llamadas adicionales a `required_consumption` en
`cart/service.py` (chequeo preventivo al agregar/actualizar el carrito). Frontend: 1 tipo de
dominio nuevo (`OptionSelectionPayload`), ~4 interfaces existentes extendidas
(`CartItemPayload`, `CartItemOption`, `MenuOption`/`MenuOptionGroup`, `ProductSelection`), 1
componente restructurado (`product-select.component.ts`,
toggle → stepper condicional), 1 servicio de lookup extendido (`menu-lookup.ts`), 6 puntos de
renderizado con el mismo cambio mecánico de una línea.

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| Principio | Evaluación | Estado |
|---|---|---|
| **I. Las Nuevas Funcionalidades Nacen de un Spec** | `spec.md` existe, aprobado, 6 historias priorizadas, 13 FRs, sesión de Clarifications (2026-09-01) que resolvió las 2 decisiones de diseño (topes configurables, sin mínimo posible) antes de este plan. | PASS |
| **II. El Comportamiento Existente Sigue Protegido** | El único cambio de comportamiento sobre código ya vigente es el cambio de forma del payload (`option_ids`→`options`), explícitamente autorizado por `spec.md` como parte de esta funcionalidad nueva — no es la corrección de una regla heredada, es la extensión necesaria para que "cantidad" pueda expresarse. Para todo grupo "conteo" (el 100% del catálogo hasta que un admin elija "cantidad"), el resultado de precio/consumo/validación es idéntico al de hoy (research.md Decisión 2/3, "cero cambio de comportamiento"). | PASS |
| **III. Los Characterization Tests Protegen el Comportamiento Heredado** | Ningún characterization test existente asume que `compute_line_price`/`plan_line_consumption`/`validate_option_selection` reciben `list[Option]` en vez de `list[ChosenOption]` como tipo — es un cambio de firma interna, no de comportamiento observable para pruebas que ya pasan un `Option` por vía correcta (se envuelve en `ChosenOption(option, 1)` donde haga falta). Cualquier test que construya `CartItemIn`/línea de orden a mano con `option_ids` se actualiza explícitamente a la forma nueva, citando esta spec (Fase de tasks). | PASS |
| **IV. Los Nuevos Specs Pueden Introducir Nuevo Comportamiento** | El criterio de éxito es conformidad con `spec.md` (SC-001 a SC-006), no equivalencia total con el pasado — coherente con que esta spec introduce un modo de selección que no existía. | PASS |
| **V. Nuevas Funcionalidades Antes que Refactorizaciones Oportunistas** | Generalizar `compute_line_price`/`plan_line_consumption` a `ChosenOption` no es una refactorización ajena — es el mecanismo mínimo necesario para que "cantidad" exista sin duplicar el motor de precio/consumo (research.md Decisión 2). No se toca ningún otro cálculo (promociones, descuentos) fuera de este cambio de tipo. | PASS |
| **VI. Evolución Incremental** | Las seis historias de usuario son independientemente verificables: US1 (declarar modo) no depende de US2 (selección del cliente); US4 (topes) y US5 (visualización) son extensiones opcionales sobre US2/US3 ya funcionando. `/speckit-tasks` debe organizarlas en fases separadas, igual que spec 063/064. | PASS |
| **VII. Compatibilidad con Datos Históricos** | Ningún `Sale`/`Payment`/`Invoice` ya emitido se toca. La migración asigna `quantity=1` a todo dato histórico por ser el único valor posible antes de esta spec — no es una aproximación, es el hecho histórico correcto (data-model.md). | PASS |
| **VIII. Evolución del Modelo de Datos** | `data-model.md` especifica las columnas nuevas (3 en `option_groups`, 1 en cada tabla de unión), su tipo, nulabilidad, default, compatibilidad con datos existentes (sin backfill por consulta — el default ya es el valor correcto) y estrategia de rollback (`downgrade` reversible, columnas nuevas sin dependencias cruzadas que impidan un `DROP COLUMN` limpio). | PASS |
| **IX. Dependencias Nuevas Permitidas con Justificación** | No aplica — cero dependencias nuevas en `requirements.txt`/`package.json`. | N/A |
| **X. Verificación Obligatoria** | Cada historia de usuario tiene su "Independent Test" en `spec.md`; `quickstart.md` los traduce a pasos ejecutables, incluyendo la verificación explícita de que el modo "conteo" rechaza `quantity>1` (no solo que "cantidad" funcione). | PASS |
| **XI. Decisiones de Negocio Frente a Decisiones Técnicas** | Las dos decisiones de negocio (topes configurables por opción y por grupo; "cantidad" siempre opcional, sin mínimo) ya quedaron registradas en `spec.md` (Clarifications) antes de este plan. Las decisiones de este plan son técnicas: unificar el payload en pares `(opción, cantidad)` en vez de mantener dos campos paralelos (Decisión 1), introducir `ChosenOption` (Decisión 2), y no agregar override por variante para los topes (Decisión 6, explícitamente diferido por no estar pedido). Ninguna decisión técnica cambia qué puede hacer el usuario más allá de lo ya definido en el spec. | PASS |
| **XII. Trazabilidad** | Cadena completa: `spec.md` (Necesidad+Spec+Clarifications) → este `plan.md`/`research.md`/`data-model.md`/`contracts/` (Decisión técnica) → `tasks.md` (Fase 2, no generada por este comando) → implementación → tests nuevos/ajustados por historia → `quickstart.md` (Verificación). | PASS |
| **XIII. Todo en Español de Colombia** | Este plan y todos sus artefactos se escriben en español de Colombia, igual que `spec.md`. | PASS |

Sin violaciones. La tabla de Complexity Tracking al final de este documento queda vacía — el
tamaño real de la superficie tocada (Technical Context, Scale/Scope) se gestiona dividiendo en
historias de usuario independientemente verificables (Principio VI), no como una excepción a
justificar.

**Re-chequeo post-diseño (Fase 1)**: `research.md`, `data-model.md` y `contracts/` no introdujeron
ninguna entidad, dependencia ni decisión que contradiga la tabla anterior. Un punto merece mención
explícita: la investigación de esta spec (sesión `/speckit-clarify`) especuló inicialmente que
haría falta soltar los `UniqueConstraint` de `CartItemOption`/`OrderItemOption` para permitir
repetir una opción — el diseño final (Decisión 4) no los toca en absoluto, una columna `quantity`
en la misma fila logra el mismo resultado sin ese riesgo. Gates siguen en PASS.

**Re-chequeo tras `/speckit-analyze`**: el análisis cruzado spec/plan/tasks encontró que este plan
(y `tasks.md` en su primera versión) solo generalizaban `plan_line_consumption` dentro de
`catalog_engine/consumption.py`, sin cubrir sus dos consumidores directos en el mismo módulo
(`required_consumption`, `ensure_lines_consume_inventory`) ni la cadena real de deducción de
inventario que los invoca (`app/api/v1/orders/consumption.py` — 6 funciones — y
`app/api/v1/sales/consumption.py`, que lee el snapshot JSONB de `SaleItem.options`) ni tres
llamadas adicionales a `required_consumption` en `cart/service.py`. Sin esas piezas, confirmar una
orden o cobrar una venta de mostrador con un grupo "cantidad" habría fallado en tiempo de
ejecución — un riesgo real para el Principio X (Verificación Obligatoria) de US3. Corregido en
Scale/Scope arriba y en `tasks.md` (T033-T036 nuevas). Gates siguen en PASS tras la corrección.

## Project Structure

### Documentation (this feature)

```text
specs/065-opciones-por-cantidad/
├── plan.md              # Este fichero (/speckit-plan)
├── research.md          # Fase 0 — decisiones técnicas y alternativas descartadas
├── data-model.md         # Fase 1 — columnas nuevas, ChosenOption, migración
├── quickstart.md         # Fase 1 — validación ejecutable por historia de usuario
├── contracts/            # Fase 1 — contratos de los endpoints y del payload compartido
│   ├── option-group-selection-mode.md
│   └── cart-order-option-quantity.md
└── tasks.md              # Fase 2 (/speckit-tasks — NO se crea en este comando)
```

### Source Code (repositorios sibling de `pos-specs`)

Esta spec vive en `pos-specs`, pero el código que describe está en `../pos-backend` y
`../pos-heladeria` (Constitución §Alcance). Rutas relativas a la raíz de cada repo. **Setup**: parte
de la rama `064-grupos-opciones-precio-inventario` en ambos repos (research.md, "Nota sobre
numeración de ramas"), no de `develop`, salvo que ya esté integrada para entonces.

```text
# pos-backend
app/models/
├── option_group.py                    # MODIFICADO — `selection_mode`, `max_quantity_per_option`,
                                          `max_total_quantity` (data-model.md)
├── cart_item.py                       # MODIFICADO — `CartItemOption.quantity`
└── order_item.py                      # MODIFICADO — `OrderItemOption.quantity`

alembic/versions/
└── <hex>_option_group_selection_mode_and_item_option_quantity.py  # NUEVO —
                                          `down_revision = '68326ed66ebf'` (head real confirmado,
                                          research.md Decisión 5), mismo patrón `_has_table` +
                                          `@for_each_tenant_schema` que las migraciones de
                                          spec 027/063

app/catalog_engine/
├── core.py                            # MODIFICADO — nuevo `ChosenOption(NamedTuple)`;
                                          `compute_line_price` generaliza a `× chosen.quantity`
                                          (research.md Decisión 2)
├── pricing.py                         # MODIFICADO — `load_valid_options` produce
                                          `list[ChosenOption]` desde `list[OptionSelectionIn]`,
                                          rechaza `option_id` repetido (422);
                                          `validate_option_selection` gana la rama "cantidad"
                                          (topes) y el rechazo de `quantity>1` en "conteo"
                                          (research.md Decisión 3)
└── consumption.py                     # MODIFICADO — `plan_line_consumption` generaliza a
                                          `per_unit × qty × chosen.quantity` (research.md
                                          Decisión 2); `required_consumption` y
                                          `ensure_lines_consume_inventory` (mismo fichero, sus dos
                                          consumidores directos) generalizan su propia firma a
                                          `Sequence[ChosenOption]` en lockstep — hallazgo de
                                          `/speckit-analyze`, sin esto la deducción real de
                                          inventario queda con un tipo desalineado

app/api/v1/orders/
└── consumption.py                     # MODIFICADO (hallazgo de `/speckit-analyze`) — las 6
                                          funciones (`ensure_consumes_inventory`,
                                          `lock_consumption`, `deduct_order_items`,
                                          `reverse_order_items`, `deduct_order_item`,
                                          `reverse_order_item`) cambian su tipado de
                                          `list[Option]` a `list[ChosenOption]`

app/api/v1/sales/
└── consumption.py                     # MODIFICADO (hallazgo de `/speckit-analyze`) —
                                          `_sale_item_options` pasa de devolver `list[Option]` a
                                          `list[ChosenOption]`, leyendo la clave `"quantity"` del
                                          snapshot JSONB de `SaleItem.options` (ya agregada por
                                          `sales/service.py`, ver más abajo); `deduct_sale` ajusta
                                          su tipado en consecuencia

app/api/v1/catalog/
└── schemas.py                         # MODIFICADO — `OptionGroupCreate.selection_mode` (con
                                          default `'conteo'`), `max_quantity_per_option`,
                                          `max_total_quantity`; mismo trío en `OptionGroupUpdate`/
                                          `OptionGroupResponse`; nuevo `OptionSelectionIn`
                                          (contracts/option-group-selection-mode.md,
                                          cart-order-option-quantity.md)

app/api/v1/cart/
├── schemas.py                         # MODIFICADO — `CartItemIn.option_ids` → `options:
                                          list[OptionSelectionIn]` (mismo cambio en
                                          `CartItemUpdate`); `CartItemOptionResponse` gana
                                          `quantity`
└── service.py                         # MODIFICADO (omitido en la versión original de este
                                          plan, corregido tras `/speckit-analyze`) — los dos call
                                          sites de `load_valid_options`/`compute_line_price` (alta
                                          y actualización de línea) pasan `data.options`; las dos
                                          construcciones de `CartItemOption` persisten `quantity`;
                                          las tres llamadas a `required_consumption` (chequeo
                                          preventivo de stock al mutar el carrito) reciben
                                          `list[ChosenOption]`; la copia sin recalcular de
                                          `POST /cart/submit` propaga `quantity` de
                                          `CartItemOption` a `OrderItemOption`

app/api/v1/orders/
├── schemas.py                         # MODIFICADO — misma sustitución `option_ids`→`options`
                                          en el schema de línea de `POST /orders`
├── service.py                         # MODIFICADO — `create_order` pasa `options` (ya
                                          `list[ChosenOption]` vía `load_valid_options`) a
                                          `compute_line_price`/persistencia de
                                          `OrderItemOption.quantity`
└── consolidation.py                   # MODIFICADO — `add_item_to_table` mismo cambio de forma
                                          (camino real de `load_valid_options`); `consolidate_table`
                                          (camino de copia desde el carrito sin recalcular, hallazgo
                                          de `/speckit-tasks`) propaga `quantity` de
                                          `CartItemOption` a `OrderItemOption` igual que
                                          `cart/service.py::submit`

app/api/v1/sales/
├── schemas.py                         # MODIFICADO — misma sustitución en la línea de venta de
                                          mostrador
├── service.py                         # MODIFICADO — `options_snapshot` gana `"quantity"` por
                                          dict (contracts/cart-order-option-quantity.md)
└── builder.py                         # SIN CAMBIO — `build_sale`/`SaleLine` no inspeccionan el
                                          contenido de `options`, solo lo persisten (FR-011)

app/api/v1/orders/checkout.py          # MODIFICADO — `order_sale_lines`/`_item_options` leen
                                          `OrderItemOption.quantity` ya persistido, sin recalcular
                                          nada nuevo; el snapshot de `options` para `SaleLine` gana
                                          `"quantity"`

app/characterization_tests/
├── test_catalog_option_groups_pricing_type.py  # SIN CAMBIO — sigue cubriendo `pricing_type`
├── test_catalog_option_groups_selection_mode.py  # NUEVO — casos de "cantidad": precio, consumo,
                                          topes, rechazo de `quantity>1` en "conteo"
└── (tests existentes que construyan `option_ids` a mano)  # MODIFICADO — se actualizan a
                                          `options: [...]`, citando esta spec (Principio III)

# pos-heladeria
src/app/modules/products/interfaces/
└── product.interface.ts               # MODIFICADO — `OptionGroup.selection_mode`,
                                          `max_quantity_per_option`, `max_total_quantity`;
                                          `MenuOptionGroup.selection_mode`; `MenuOption` sin cambio
                                          propio (la cantidad vive en la selección, no en la
                                          opción del catálogo)

src/app/modules/tables/interfaces/
└── diner.interface.ts                 # MODIFICADO — `CartItemPayload.option_ids` →
                                          `options: OptionSelectionPayload[]`;
                                          `CartItemOption.quantity` (contracts/
                                          cart-order-option-quantity.md)

src/app/modules/tables/components/
└── product-select.component.ts        # MODIFICADO — `selected: Record<groupId, string[]>` →
                                          `Record<groupId, Record<optionId, number>>`; rama de
                                          plantilla nueva (stepper) cuando
                                          `group.selection_mode === 'cantidad'`; `chosenCount`/
                                          `isComplete`/`groupHint`/`requiredCount` se ramifican
                                          (research.md Decisión 8)

src/app/modules/tables/services/
├── dining-cart.service.ts             # MODIFICADO — construye `options: OptionSelectionPayload[]`
                                          en vez de `option_ids: string[]`
└── menu-lookup.ts                     # MODIFICADO — nuevo método `optionLabelWithQuantity`
                                          (research.md Decisión 9)

src/app/modules/tables/components/     # Los siguientes 6 puntos cambian su
├── pos-terminal.store.ts (línea ~794) #   `.map(o => lk.optionLabel(o.option_id))` por
├── order-detail.component.ts (~144)   #   `.map(o => lk.optionLabelWithQuantity(o.option_id,
├── payment-validation-block.component.ts (~131)  #   o.quantity ?? 1))` — cambio mecánico
├── cart.component.ts (~30-32)         #   idéntico en los 6 (research.md Decisión 9)
└── (ver también)
src/app/modules/products/pages/public-menu.component.ts (~1095)
src/app/shared/receipt/receipt.util.ts (~88, ~216-218)
```

**Structure Decision**: Web application con dos repositorios (backend FastAPI + frontend Angular),
estructura ya establecida por specs previas — esta spec no agrega ningún directorio ni módulo
nuevo, modifica archivos existentes y agrega 1 migración + 1 archivo de test backend nuevo.

## Complexity Tracking

*Sin violaciones que justificar — tabla vacía.*
