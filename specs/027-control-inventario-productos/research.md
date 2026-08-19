# Research: Control de Inventario por Producto (Switch de Insumos)

No quedó ningún `NEEDS CLARIFICATION` en el Technical Context del plan — las 2 clarificaciones de
negocio ya se resolvieron en `spec.md` (sesión `/speckit-clarify`, 2026-08-19) y el resto de
incógnitas era puramente técnico, resuelto leyendo directamente `pos-backend`/`pos-heladeria`. Este
documento registra las decisiones de diseño y las alternativas descartadas.

## Decisión 1 — La exención de FR-005 se inyecta en `plan_line_consumption`, no solo en el guard

- **Decisión**: `app/catalog_engine/consumption.py` es, por diseño propio del módulo (docstring,
  líneas 1-9: "Antes esto vivía triplicado... Por eso hay un solo sitio que decide qué se consume"),
  el único lugar que calcula qué descuenta una línea. `plan_line_consumption` (líneas 89-129) es esa
  función única — la llaman directamente `deduct_order_item`/`reverse_order_item`
  (`app/api/v1/orders/consumption.py`) y `deduct_sale` (`app/api/v1/sales/consumption.py`) para el
  descuento/reversa real, y también `required_consumption` (líneas 134-144, que solo la envuelve
  para agregar por insumo) para el chequeo de disponibilidad y el guard. Se agrega una función
  auxiliar `_tracks_inventory(db, variant_id) -> bool` (junto a `variant_label`, línea ~146) que hace
  el mismo tipo de JOIN `ProductVariant → Product` ya usado ahí, y `plan_line_consumption` la
  consulta como primera línea de su cuerpo, devolviendo `[]` de inmediato cuando es `False` — antes
  de tocar `load_recipe`/`load_variant_groups`.
- **Rationale**: parchar únicamente `ensure_lines_consume_inventory` (el guard) no habría bastado.
  Por FR-008/Edge Cases de `spec.md`, un producto con el switch apagado puede perfectamente conservar
  insumos ya guardados (el switch no los borra). Si la exención solo viviera en el guard,
  `required_consumption` seguiría siendo no-vacío para esa variante (porque `plan_line_consumption`
  no sabría nada del switch) → el guard dejaría pasar la venta (nada que rechazar) → pero
  `deduct_order_item`/`deduct_sale`, que llaman `plan_line_consumption` **directamente**, sí
  generarían movimientos de inventario reales — violando exactamente la cláusula de FR-005 ("NO DEBE
  generar ningún movimiento de inventario"). Inyectar la exención en la función que de verdad decide
  qué se consume corrige los tres llamadores reales a la vez, sin tocar sus firmas ni su lógica.
- **Alternatives considered**: (a) agregar un parámetro `tracks_inventory: bool` a
  `plan_line_consumption` que cada llamador deba resolver y pasar — descartado: obliga a tocar las
  firmas de `deduct_order_item`, `reverse_order_item` y `deduct_sale`, además del propio guard, para
  algo que la función ya puede resolver internamente con una consulta más, del mismo tipo que ya hace
  `variant_label`. (b) Cachear `tracks_inventory` en memoria por request — descartado como
  sobre-ingeniería no pedida por el spec; el módulo entero ya acepta un query extra por entrada
  (`load_recipe`, `load_variant_groups`, `variant_label`, `required_consumption` son todos O(N)
  consultas por línea), así que una consulta más es consistente con el estilo ya existente, no una
  regresión de performance nueva (Principio V: no optimizar lo que nadie pidió).

## Decisión 2 — El guard también necesita su propio chequeo, independiente del de Decisión 1

- **Decisión**: `ensure_lines_consume_inventory` (líneas 156-217) no solo confía en que
  `required_consumption` quede vacío — tiene una clasificación secundaria (líneas 176-179) que
  consulta `load_recipe`/`group_discounts` **directamente** contra la base de datos para distinguir
  "sin nada configurado" (`sin_receta`, bloquea con un mensaje) de "configurado pero no elegido"
  (`sin_eleccion`, bloquea con otro mensaje). Esa clasificación bypasea a `plan_line_consumption` por
  completo, así que el arreglo de Decisión 1 no la cubre: un producto con switch apagado que **sí**
  tiene insumos guardados de antes (Historia 3 del spec) haría que `configurada = True` →
  `sin_eleccion.append(...)` → `409` — bloqueando exactamente la venta que FR-005 exige permitir. Por
  eso el bucle principal de `ensure_lines_consume_inventory` (línea 179) agrega, como primera
  condición de cada iteración, el mismo chequeo `_tracks_inventory(db, variant_id)` de Decisión 1: si
  es `False`, `continue` de inmediato, sin llegar ni a `required_consumption` ni a la clasificación
  secundaria.
- **Rationale**: son dos caminos de código independientes dentro del mismo módulo (uno decide qué se
  descuenta, el otro decide si rechazar la venta) — exentar uno sin el otro dejaría una mitad del
  problema sin resolver. Verificado leyendo el cuerpo completo de la función, no solo su firma.
- **Alternatives considered**: hacer que `required_consumption` "mienta" devolviendo un valor no
  vacío ficticio para evitar que el guard entre a la clasificación secundaria — descartado, es más
  frágil (dos funciones necesitarían coordinarse con un valor sentinela) que simplemente repetir la
  misma consulta corta al principio de ambas, que además dice explícitamente su intención.

## Decisión 3 — El default de la columna en el modelo ORM es `True`, aunque FR-001 pida `False` en el formulario

- **Decisión**: `Product.tracks_inventory` se agrega con `default=True, server_default="true"` a
  nivel de modelo SQLAlchemy (mismo patrón que `available`, `app/models/product.py:37-39`). El
  default `False` que pide FR-001 ("apagado por defecto en todo producto nuevo") se implementa
  **únicamente** en el schema Pydantic `ProductCreate` (`tracks_inventory: bool = False`,
  `app/api/v1/products/schemas.py:14-23`) — el único punto por el que pasa la creación desde el
  formulario nuevo, vía `POST /products` → `ProductService.create_product`
  (`app/api/v1/products/service.py:41-65`), que ya construye `Product(...)` pasando cada campo del
  body explícitamente (línea ~46-53) — nunca deja que el ORM aplique su propio default en ese camino.
- **Rationale**: es la decisión más importante de este research, porque el error opuesto (poner
  `default=False` también en el modelo) rompería en silencio **toda** la suite de characterization
  tests de spec 003 (`test_catalog_consumption_plan.py`, `golden_master_core.py`, y cualquier otro
  test que construya `Product(...)` directamente por ORM sin conocer este campo nuevo, que es la
  inmensa mayoría — el concepto no existía antes de esta spec). Con `default=False` en el modelo,
  cualquier fixture existente que hoy construye una variante con receta y espera que descuente
  inventario dejaría de descontar nada, sin que el test fallara por una razón obvia relacionada con lo
  que realmente está probando — exactamente el tipo de regresión silenciosa que Principio III prohíbe.
  Con `default=True` en el modelo, cualquier código (tests incluidos) que no sepa nada de este campo
  nuevo se comporta exactamente igual que antes de esta spec — el comportamiento actual es sagrado por
  defecto (Principio II) para todo código que no pase explícitamente por el formulario nuevo.
- **Alternatives considered**: usar el mismo valor (`False`) en ambos niveles, "para que sea
  coherente" — descartado tras trazar la cadena real de llamadas: la coherencia superficial (un solo
  valor) habría escondido una regresión real y extensa en la suite de tests protegidos, que es
  justamente lo que la constitución de este proyecto prioriza por encima de la simetría cosmética del
  código.

## Decisión 4 — Migración: `server_default="true"` + backfill dirigido por `UPDATE ... EXISTS`

- **Decisión**: nueva migración `alembic/versions/<rev>_products_tracks_inventory.py`,
  `down_revision = 'd2e3f4a5b6c7'` (head actual, `d2e3f4a5b6c7_active_order_per_participant.py`),
  siguiendo el esqueleto exacto de `f5a6b7c8d9e0_availability_change_partial_count.py` (mismo patrón
  para `products.available`) y `b8c9d0e1f2a3_option_group_active.py`: decorador
  `@for_each_tenant_schema`, guard `_has_table(schema, "products")`, y:
  ```python
  op.add_column(
      "products",
      sa.Column("tracks_inventory", sa.Boolean(), nullable=False, server_default="true"),
      schema=schema,
  )
  op.execute(f"""
      UPDATE {schema}.products p
      SET tracks_inventory = EXISTS (
          SELECT 1 FROM {schema}.product_variants pv
          WHERE pv.product_id = p.id AND pv.active = true AND (
              EXISTS (
                  SELECT 1 FROM {schema}.recipe_items ri
                  WHERE ri.product_variant_id = pv.id
              )
              OR EXISTS (
                  SELECT 1 FROM {schema}.variant_option_groups vog
                  WHERE vog.product_variant_id = pv.id AND vog.quantity_per_option > 0
              )
              OR EXISTS (
                  SELECT 1 FROM {schema}.variant_option_groups vog
                  JOIN {schema}.options o ON o.option_group_id = vog.option_group_id
                  WHERE vog.product_variant_id = pv.id
                    AND o.active = true
                    AND o.inventory_item_id IS NOT NULL
                    AND o.item_quantity > 0
              )
          )
      )
  """)
  ```
  `downgrade` hace `op.drop_column("products", "tracks_inventory", schema=schema)` — mismo patrón que
  `b8c9d0e1f2a3` (reversible sin manejo especial: perder la columna no destruye ningún otro dato).
- **Rationale**: la condición `EXISTS` del `UPDATE` replica exactamente, en SQL, la lógica de
  `load_recipe` (`RecipeItem` existe para la variante — su `CHECK ck_recipe_item_qty_positive`
  garantiza `quantity > 0` siempre, así que no hace falta repetir esa condición) y `group_discounts`
  (`app/catalog_engine/consumption.py:70-88`: `quantity_per_option > 0` manda; si no, respalda que
  alguna opción activa del grupo tenga `inventory_item_id` e `item_quantity > 0`) — las dos fuentes
  de consumo que hoy determinan si una variante "ya superaba" `RN-CAT-34` (FR-010/FR-011 del spec).
  Restringir a `product_variants.active = true` es una precisión deliberada del backfill (no del
  guard, que nunca recibe variantes inactivas para vender de todas formas): una presentación
  desactivada con receta vieja no debería, por sí sola, hacer que el producto migre con el switch
  encendido si ninguna presentación **vendible** tiene nada configurado.
- **Alternatives considered**: escribir un script de backfill en Python que reutilice
  `load_recipe`/`group_discounts` importados desde `app.catalog_engine.consumption` dentro de la
  propia migración — descartado siguiendo el precedente ya establecido del proyecto
  (`a1b2c3d4e5f6_promotions_refactor.py:129-130`, backfill vía `op.execute(UPDATE ... CASE WHEN
  ...)`, sin importar código de aplicación): las migraciones de este repo se mantienen
  autocontenidas en SQL para no depender de que el código de `app/` no cambie de forma incompatible
  en el futuro y rompa una migración ya aplicada en producción.
- **Verificación adicional recomendada** (no es parte de la migración en sí, va a `tasks.md`): antes
  de aplicar en producción, correr un script de un solo uso que compare, para cada `ProductVariant`
  activa de un tenant real, el resultado de la subconsulta `EXISTS` de arriba contra el resultado real
  de invocar `load_recipe`/`group_discounts` (las funciones Python ya probadas), y confirmar 0
  discrepancias — es la única forma de tener certeza de que SC-003/SC-004 se cumplen exactamente,
  dado que la lógica se duplicó intencionalmente en SQL (Decisión 4) en vez de reutilizarse.

## Decisión 5 — Characterization tests: `test_catalog_consumption_plan.py` se extiende, no se reescribe

- **Decisión**: el módulo CONGELA (`app/characterization_tests/test_catalog_consumption_plan.py`,
  docstring líneas 1-7, clase `EnsureLinesConsumeInventoryTests` líneas 180-234) se extiende con
  casos nuevos para `tracks_inventory=False` (Historia 1: producto sin insumos y sin receta vende sin
  bloqueo; Historia 3: producto con insumos guardados pero switch apagado tampoco bloquea ni genera
  `ConsumptionLine`), citando esta spec (FR-005) como autorización en el mismo commit que las agrega,
  conforme al Principio III. Los 4 tests ya existentes
  (`test_rn_cat_34_variante_sin_receta_ni_grupo_bloquea_con_409_sin_receta`,
  `test_rn_cat_35_discrepancia_grupo_opcional_unica_fuente_sin_elegir_bloquea_sin_eleccion`,
  `test_variante_con_receta_fija_y_grupo_opcional_sin_elegir_no_bloquea`,
  `test_rn_cat_34_variante_con_grupo_obligatorio_que_descuenta_pero_ninguna_opcion_elegida`) **no se
  modifican** — sus fixtures, al construir `Product(...)` sin especificar `tracks_inventory`, heredan
  el default `True` del modelo (Decisión 3), así que siguen ejerciendo exactamente el mismo camino de
  código que antes de esta spec, sin necesitar ningún cambio.
- **Rationale**: es la prueba concreta de que Decisión 3 funciona — si el default del modelo fuera
  `False`, estos 4 tests dejarían de pasar (sus variantes con receta configurada dejarían de generar
  consumo) y esta spec estaría violando Principio III en el mismo cambio que dice protegerlo.
- **Alternatives considered**: ninguna — esta decisión es la consecuencia directa, no una alternativa
  evaluada aparte, de Decisión 3.
- **Cobertura indirecta a verificar, sin modificar**: `golden_master_core.py` (líneas 63, 90, 126,
  159, 189, 215, 279, 301, con comentarios citando `RN-CAT-34` en 95, 219, 282, 304) construye
  variantes con receta ya configurada — debe seguir en verde sin cambios, por la misma razón de
  Decisión 3. `catalog_engine_equivalence_gate.py` está archivado (docstring propio: "ARCHIVADO tras
  la Historia 3") y no se toca.

## Decisión 6 — Frontend: el switch reutiliza el patrón real ya usado para "hasSizes", a nivel de `draft()` (producto)

- **Decisión**: se agrega `tracks_inventory: boolean` a `ProductDraft`
  (`pos-heladeria/src/app/modules/products/interfaces/product.interface.ts:268-281`, junto a
  `hasSizes`, que ya vive al mismo nivel — el del producto, no el de cada `VariantDraft`) y se
  replica el markup exacto del switch de `hasSizes`
  (`product-form.component.ts:145-150`: `role="switch"`, `[attr.aria-checked]`, círculo deslizante
  `bg-indigo-600`/`bg-gray-300`) con un nuevo método `toggleTracksInventory()` (junto a
  `toggleHasSizes()`, línea 734) que usa el setter genérico `setField(...)` ya existente (líneas 105,
  110, 120, 128) para "Datos generales", consistente con que `tracks_inventory` es un campo de
  producto, no de variante (FR-007).
- **Rationale**: el proyecto no tiene `mat-slide-toggle` ni un componente switch reutilizable (ya
  confirmado) — pero **sí** tiene ya un switch real, accesible y con el estilo visual correcto
  funcionando en producción a un clic de distancia en el mismo archivo; copiarlo es la opción de
  menor riesgo y mayor consistencia visual (Principio V: no inventar un componente nuevo cuando el
  patrón ya existe en el mismo formulario).
- **Alternatives considered**: extraer un componente `<app-switch>` reutilizable a partir de este
  patrón — fuera de alcance del spec (`Out of Scope`: "Definir el diseño visual concreto del
  componente switch... se resuelve en la fase de planeación" no pide una extracción de componente;
  spec 026 ya estableció el precedente de no introducir componentes nuevos cuando el efecto se logra
  modificando los ya existentes, Principio V).

## Decisión 7 — Envolver "Insumos fijos" Y "Sabores a elegir" con la misma condición

- **Decisión**: los dos bloques ya existentes de `product-form.component.ts` — "Insumos fijos"
  (líneas 218-242, receta fija) y "Sabores a elegir" (líneas 244-350, grupos de opciones) — quedan
  ambos envueltos en `@if (draft().tracks_inventory) { ... } @else { <p>mensaje de sección
  deshabilitada</p> }`, dentro del `@if (activeVariant(); as av)` que ya los contiene.
- **Rationale**: FR-002/FR-003 hablan de "la sección de insumos" en singular, y la spec (Assumptions)
  ya aclaró explícitamente que ese término cubre ambos mecanismos de descuento (receta fija y grupos
  de opciones) — coincide 1:1 con las dos fuentes que evalúa `ensure_lines_consume_inventory`
  (Decisión 1/2): dejar uno de los dos bloques editable con el switch apagado permitiría configurar
  algo que, aun así, nunca se aplicaría al vender (Decisión 1), una inconsistencia de UI que
  confundiría más de lo que ayudaría.
- **Alternatives considered**: ocultar los bloques por completo (`@if`) en vez de deshabilitarlos
  visualmente mostrando un mensaje — se prefiere mostrar un mensaje corto ("Activa 'Maneja
  inventario' para configurar insumos") en vez de que la sección desaparezca sin explicación, más
  coherente con FR-003 ("deshabilitada", no "ausente") y con que el usuario entienda por qué no ve
  los campos que sabe que existen para otros productos.

## Decisión 8 — FR-013 (aviso al guardar) reutiliza el patrón ámbar ya existente; no es un banner nuevo desde cero

- **Decisión**: el formulario ya tiene una advertencia inline por grupo de opciones
  (`groupBreakdown(g)`, líneas 286 y 296: `<p class="text-xs text-amber-700 mt-1.5">⚠ {{ w }}</p>` y
  `<span class="text-amber-700 font-medium">· ⚠ {{ bd.missing }} sin insumo</span>`, dentro de un
  contenedor `border-amber-200 bg-amber-50/40`, línea 254) — es la misma familia visual de aviso que
  pide FR-013/SC-007, solo que hoy actúa por grupo de opciones, no como resumen a nivel de producto.
  Se agrega un banner nuevo del mismo estilo (`border-amber-200 bg-amber-50/40`, ícono `⚠` como texto
  plano) justo debajo del switch, visible cuando `draft().tracks_inventory` es `true` y ninguna
  presentación tiene receta (`av.recipe.length > 0`) ni ningún grupo que efectivamente descuente —
  evaluado en el mismo momento en que se intenta guardar (`onSubmit`/método equivalente ya existente
  en el componente), no solo de forma reactiva mientras se edita.
- **Rationale**: reutilizar la paleta/ícono ya usado en el mismo formulario evita introducir un
  segundo lenguaje visual de advertencia (Principio V); la condición ("ninguna presentación con algo
  configurado") es exactamente la misma que evalúa el backend en Decisión 1/2, replicada del lado del
  cliente sobre el `draft()` ya cargado en memoria — no requiere ninguna llamada nueva al backend.
- **Alternatives considered**: disparar una llamada al backend para preguntar "¿esta configuración ya
  pasaría el guard?" antes de guardar — descartado por innecesario: el mismo criterio
  (`recipe.length > 0` o algún `optionGroups` con consumo) ya es 100% derivable del `draft()` que el
  formulario ya tiene en memoria, sin ida y vuelta de red.

## Decisión 9 — FR-014 (confirmación al apagar) reutiliza `ConfirmService`, no un modal nuevo

- **Decisión**: `toggleTracksInventory()` (Decisión 6), al pasar de `true` a `false`, primero revisa
  si el producto tiene algo configurado (mismo criterio de Decisión 8: algún `av.recipe.length > 0` o
  grupo con consumo, sobre **todas** las variantes del `draft()`, no solo la activa) y, si lo tiene,
  llama `await this.confirm.ask({ tone: 'danger', ... })` — el servicio ya existente
  `pos-heladeria/src/app/shared/feedback/confirm.service.ts:21-38`, con el mismo patrón usado hoy en
  `option-groups-page.component.ts:229`, `pos-terminal.store.ts:923,941,1027` y
  `promotions-page.component.ts:2323,2447`. Si el usuario cancela, `setField('tracks_inventory',
  true)` nunca se ejecuta (el toggle no cambia nada) — replicando el mismo patrón "await + solo
  aplicar si `ok`" ya usado en esos otros call sites.
- **Rationale**: el proyecto ya resolvió "modal de confirmación reutilizable" como problema general;
  reimplementarlo sería una regresión de consistencia además de trabajo no pedido (Principio V).
- **Alternatives considered**: un `window.confirm()` nativo del navegador — descartado, rompe el
  estilo visual del resto de la aplicación y es el tipo de atajo que el propio `ConfirmService` existe
  para evitar.

## Decisión 10 — Sin cambios a `orders/consumption.py` ni `sales/consumption.py`

- **Decisión**: ningún archivo fuera de `app/catalog_engine/consumption.py` (backend) necesita
  tocarse para que FR-001 a FR-012 funcionen — `deduct_order_item`, `reverse_order_item` y
  `deduct_sale` siguen llamando `plan_line_consumption`/`ensure_lines_consume_inventory` exactamente
  igual que hoy; el cambio de comportamiento vive enteramente dentro de las dos funciones ya
  identificadas (Decisión 1/2).
- **Rationale**: consecuencia directa de que la exención se inyectó en el punto único de decisión del
  módulo (Decisión 1) — es la confirmación de que el diseño logra el mínimo blast radius posible
  (Principio VI).
