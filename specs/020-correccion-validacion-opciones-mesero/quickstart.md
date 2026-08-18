# Quickstart: validar la corrección de `add_item_to_table` (A-04)

Guía de ejecución para comprobar que la corrección cumple su contrato. No repite firmas ni tablas
ya detalladas en [data-model.md](./data-model.md) y
[contracts/add-item-to-table-endpoint.md](./contracts/add-item-to-table-endpoint.md) — solo enlaza
a ellas.

**Prerequisitos**: entorno virtual de `pos-backend` activado (Python 3.14), ejecutado desde la raíz
de `../pos-backend` (sibling de este repo `pos-specs`).

```bash
cd ../pos-backend
source env/bin/activate
```

No hace falta PostgreSQL real: `app/characterization_tests/orders_fixtures.py` crea SQLite en
memoria (`fx.new_session()`, `fx.make_dining_table`, `fx.make_variant`, `fx.make_option_group`,
`fx.link_variant_group`, `fx.make_user_double`).

## Paso 1 — Confirmar el defecto actual (antes de tocar el código)

```bash
python3 -m unittest app.characterization_tests.test_orders_consolidation -v
```

**Resultado esperado antes del fix**: el test T012
(`test_add_item_to_table_a04_omite_validacion_de_seleccion_de_opciones`) pasa en verde
**congelando el defecto** — confirma con datos reales que una variante con `min_select=1` acepta
una selección vacía sin error vía `add_item_to_table`, mientras que
`test_create_order_contraste_a04_si_valida_seleccion_de_opciones` confirma que el mismo escenario
vía `create_order` ya rechaza con `422`.

## Paso 2 — Aplicar la corrección

Editar `app/api/v1/orders/consolidation.py:199` según
[data-model.md](./data-model.md) §Entidad `add_item_to_table` y
[research.md](./research.md) Decisión 1:

```python
# Antes (A-04):
options = load_valid_options(db, data.option_ids)
# Después:
options = load_valid_options(db, data.option_ids, variant=variant)
```

Sin import nuevo — `variant` ya está en el ámbito local dos líneas arriba (línea 196). Ningún otro
cambio en la función: la expansión de combo, `deduct_order_items` y el resto del flujo quedan
intactos.

## Paso 3 — Modificar el test `CONGELA` existente (Principio II)

Actualizar `test_add_item_to_table_a04_omite_validacion_de_seleccion_de_opciones` en
`test_orders_consolidation.py:65-83` para verificar el rechazo con `422`, **citando A-04 en el
mismo commit** (research.md Decisión 2). El nombre pasa a reflejar el comportamiento corregido
(p. ej. `test_add_item_to_table_a04_valida_seleccion_de_opciones_tras_la_correccion`):

```python
def test_add_item_to_table_a04_valida_seleccion_de_opciones_tras_la_correccion(self):
    """CONGELA comportamiento corregido — A-04 (`consolidation.py:199`): una
    variante con un grupo de opciones obligatorio (`min_select=1`) y ninguna
    opción seleccionada se rechaza con 422, igual que ya hacía `create_order`
    (cita: registro-de-anomalias.md, A-04, "Tratamiento acordado")."""
    db = fx.new_session()
    table = fx.make_dining_table(db)
    _, _, variant, _ = self._seed_variant_con_receta(db)
    self._seed_grupo_obligatorio_que_descuenta(db, variant)
    db.commit()
    user = self._user()

    data = OrderItemIn(product_variant_id=variant.id, quantity=1, option_ids=[])
    with self.assertRaises(HTTPException) as ctx:
        consolidation.add_item_to_table(db, table.id, data, user)
    self.assertEqual(ctx.exception.status_code, 422)
```

`test_create_order_contraste_a04_si_valida_seleccion_de_opciones` (líneas 87-106) **no se toca** —
sigue documentando el comportamiento correcto de `create_order`, ahora ya no es "de contraste" sino
de paridad (ver Paso 4).

## Paso 4 — Test de paridad nuevo (FR-003/FR-006)

Añadir, en el mismo fichero, el caso que exige `research.md` Decisión 3:

```python
def test_add_item_to_table_y_create_order_convergen_tras_la_correccion(self):
    """Cierra A-04 (FR-003): el mismo escenario rechazado por los dos caminos
    con el mismo código, ya sin divergencia entre add_item_to_table y
    create_order."""
    db = fx.new_session()
    table = fx.make_dining_table(db)
    _, _, variant, _ = self._seed_variant_con_receta(db)
    self._seed_grupo_obligatorio_que_descuenta(db, variant)
    db.commit()
    user = self._user()

    data_add = OrderItemIn(product_variant_id=variant.id, quantity=1, option_ids=[])
    with self.assertRaises(HTTPException) as ctx_add:
        consolidation.add_item_to_table(db, table.id, data_add, user)

    data_create = OrderCreate(
        channel=OrderChannel.COUNTER,
        items=[OrderItemIn(product_variant_id=variant.id, quantity=1, option_ids=[])],
    )
    with self.assertRaises(HTTPException) as ctx_create:
        service.create_order(db, data_create, uuid4())

    self.assertEqual(ctx_add.exception.status_code, ctx_create.exception.status_code)
```

```bash
python3 -m unittest app.characterization_tests.test_orders_consolidation -v
```

**Resultado esperado tras el fix**:

- Historia 1, escenario 1 (T012 modificado): selección incompleta vía `add_item_to_table` → `422`,
  sin `OrderItem` creado.
- Historia 1, escenario 2: selección completa y correcta vía `add_item_to_table` → `201`, sin
  cambio frente a hoy.
- Historia 2: `add_item_to_table` y `create_order` rechazan el mismo escenario con el mismo código
  — test de paridad en verde.
- `test_add_item_to_table_combo_expande_componentes_a_precio_normal` (línea 234, sin cambios) sigue
  en verde — la corrección no toca la rama de combos.
- `test_create_order_contraste_a04_si_valida_seleccion_de_opciones` (sin cambios) sigue en verde.

## Paso 5 — No regresión en el mecanismo de `load_valid_options` (spec 004)

```bash
python3 -m unittest app.characterization_tests.test_catalog_line_pricing -v
```

**Resultado esperado**: `test_rn_cat_33_a04_sin_pasar_variant_load_valid_options_no_valida_nada`
sigue en verde sin modificarse — confirma que esta corrección no cambió `load_valid_options` en sí,
solo el argumento con el que la llama `add_item_to_table`.

## Verificación final — SC-001 a SC-004

```bash
python3 -m unittest app.characterization_tests.test_orders_consolidation -v
python3 -m unittest app.characterization_tests.test_catalog_line_pricing -v
python3 -m unittest app.characterization_tests.test_orders_service -v
```

Todos los tests de estos tres módulos en verde, incluidos los preexistentes sin cambios (T013,
T014-T020, mecanismo de `load_valid_options`, `create_order`) más el T012 modificado y el test de
paridad nuevo, es la señal de que la corrección está completa y no introdujo ninguna regresión en
la expansión de combos, la apertura de sesión/orden sobre la marcha, ni en el camino de
`create_order` (Principio II).
