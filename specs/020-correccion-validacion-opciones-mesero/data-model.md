# Phase 1 Data Model: Corrección de la validación de opciones en el alta directa del mesero (A-04)

Esta spec no introduce ni modifica ninguna tabla, modelo ORM ni esquema de base de datos — corrige
puramente el comportamiento de una llamada existente en tiempo de escritura (FR-005, Assumptions
de `spec.md`: "no se requiere migración de datos"). Las "entidades" relevantes aquí son de código
y de comportamiento, no de datos nuevos: los Key Entities que ya define `spec.md`, detallados a
nivel de función para guiar la corrección.

## Entidad: `add_item_to_table` (`app/api/v1/orders/consolidation.py:180-230`)

| Aspecto | Antes (defectuoso, A-04) | Después (esta spec) |
|---|---|---|
| Firma | `add_item_to_table(db, table_id, data, user) -> CustomerOrder` | Sin cambios |
| Rama combo (`data.combo_id is not None`) | `options = []` por diseño (no aplica) | Sin cambios — fuera de alcance (FR-004) |
| Rama variante normal | `options = load_valid_options(db, data.option_ids)` — sin `variant`, ninguna validación de `min_select`/`max_select`/pertenencia | `options = load_valid_options(db, data.option_ids, variant=variant)` — activa `validate_option_selection` (FR-001) |
| Selección que incumple `min_select`/`max_select` del grupo obligatorio (de menos o de más) | Se acepta: `OrderItem` creado, precio completo cobrado, inventario descontado solo de las opciones enviadas | `HTTPException(422)` antes de crear el `OrderItem` — ningún cobro ni descuento de inventario (FR-002) |
| Opción de un grupo no vinculado a la variante (`RN-CAT-32`/A-06, `STRICT_OPTION_SELECTION=False`) | Tolerada, sin bloquear | Sin cambio — sigue tolerada; ese criterio es de `validate_option_selection` (spec 004), no de este caller |
| Selección completa y válida | Se acepta, sin cambio | Sin cambio — comportamiento idéntico al de hoy |
| Ítems ya creados antes del fix | N/A | Sin recálculo — el fix no toca `OrderItem`s existentes (FR-005) |

**Dependencias del cambio** (todas ya existentes en `pos-backend`, ninguna nueva — Constitución
Principio IV no aplica):

- `app.api.v1.catalog.line_pricing.load_valid_options` — ya importado en
  `consolidation.py:27`; solo cambia el argumento con el que se invoca, no la función en sí (spec
  004, fuera de alcance).
- `variant` — ya está en el ámbito local de la rama `else` (`consolidation.py:196`,
  `get_or_404(db, ProductVariant, data.product_variant_id, ...)`), asignada dos líneas antes de la
  llamada que cambia.

## Entidad de referencia: `create_order` (`app/api/v1/orders/service.py:95-102`)

Sin cambios — es el criterio que esta corrección replica, no algo que esta spec modifique. Ya pasa
`variant=variant` a `load_valid_options` (línea 102) y ya rechaza con `422` la misma selección
incompleta. Documentado aquí solo como referencia de paridad (FR-003).

## Modelos ORM consultados (sin modificar — referencia)

- **`ProductVariant`** (`app/models/product_variant.py`): `id`, `active` — ya cargada por
  `add_item_to_table` antes del punto que cambia; sin cambios de schema.
- **`Option`** / **`OptionGroup`** / **`VariantOptionGroup`**: `min_select`, `max_select`,
  `option_group_id`, `quantity_per_option` — consultados sin cambios por
  `validate_option_selection` (`app/catalog_engine/pricing.py`, spec 004); esta delta no toca su
  definición ni su lógica, solo activa su ejecución para este caller.
- **`OrderItem`** / **`OrderItemOption`**: sin cambios de schema — el efecto observable es que,
  ante una selección inválida, ninguna fila nueva de estos modelos llega a crearse (antes: sí se
  creaba con `options=[]` o incompleto).

## Entidad: Contrato de comportamiento (Constitución, Principio II)

El conjunto que arbitra si la corrección preserva/corrige el comportamiento esperado:

- **Test modificado** (research.md Decisión 2): el `CONGELA` existente
  (`test_add_item_to_table_a04_omite_validacion_de_seleccion_de_opciones`,
  `test_orders_consolidation.py:65-83`) — se actualiza para verificar el rechazo con `422`, citando
  A-04 en el commit.
- **Test sin cambios**: `test_create_order_contraste_a04_si_valida_seleccion_de_opciones`
  (`test_orders_consolidation.py:87-106`) — ya verifica el comportamiento correcto de `create_order`,
  sigue vigente tal cual como referencia.
- **Test sin cambios**: `test_rn_cat_33_a04_sin_pasar_variant_load_valid_options_no_valida_nada`
  (`test_catalog_line_pricing.py:206-214`) — documenta el mecanismo de `load_valid_options` en sí
  (spec 004), no este caller; esta delta no cambia esa función.
- **Test nuevo** (FR-006, research.md Decisión 3): paridad exacta entre `add_item_to_table` y
  `create_order` ante el mismo escenario (`min_select=1` que descuenta inventario, selección
  vacía) — ambos deben rechazar con `422` después del fix.
- **Entrada citada del registro de anomalías**: A-04 (autoriza el cambio de comportamiento) — como
  referencia normativa en el nombre/comentario del test, no como artefacto ejecutable.
- **Sin golden master nuevo**: el alcance (un caller, una línea de cambio real) no tiene
  interacción encadenada entre varias funciones que lo justifique — mismo razonamiento que aplicó
  la spec 019 (`data-model.md` de esa spec) a un caso comparable.

## Transiciones de estado

No aplica en sentido de máquina de estados de negocio — el `OrderItem` que esta corrección puede
llegar a impedir crear ya tiene su ciclo documentado en la spec 009
(`pendiente → en_preparacion → listo`, `↘ anulado`); esta spec no le agrega ni le quita
transiciones, solo cambia si el `OrderItem` llega a existir cuando la selección de opciones que lo
acompaña es inválida.
