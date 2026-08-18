# Implementation Plan: Corrección de la validación de opciones en el alta directa del mesero (A-04)

**Branch**: `020-correccion-validacion-opciones-mesero` | **Date**: 2026-08-18 | **Spec**: [spec.md](./spec.md)

**Input**: Feature specification from `/specs/020-correccion-validacion-opciones-mesero/spec.md`

## Summary

`add_item_to_table` (`app/api/v1/orders/consolidation.py:199`) hoy llama
`load_valid_options(db, data.option_ids)` sin el parámetro `variant`, así que la validación de
`min_select`/`max_select`/pertenencia de grupo (`validate_option_selection`) nunca se ejecuta en el
único camino con botón real de la terminal del mesero — A-04. `service.create_order`
(`service.py:102`) sí pasa `variant=variant` y sí rechaza con `422` la misma selección incompleta.
Esta spec corrige la línea 199 para pasar `variant=variant`, replicando exactamente el criterio de
`create_order` (FR-001/FR-002), sin tocar `load_valid_options` en sí (spec 004, fuera de alcance)
ni ningún otro caller.

El fix ya existió una vez (`03469ca`, 2026-08-03) y se perdió en el merge de la rama de combos
(`ee94f30`, 2026-08-04) porque esa rama partió de una copia del fichero anterior a la corrección.
Existe hoy un characterization test que **CONGELA** el defecto explícitamente para este propósito
(`test_orders_consolidation.py:65`, docstring: "el fix de una línea... queda para una spec delta
posterior") — esta es esa spec delta. El Principio II exige modificar ese test citando A-04 en el
commit, no "ajustarlo" en silencio.

## Technical Context

**Language/Version**: Python 3.14 (venv de `pos-backend`, `env/pyvenv.cfg`)

**Primary Dependencies**: FastAPI + SQLAlchemy (ya en uso); ningún import nuevo —
`load_valid_options` (`app.api.v1.catalog.line_pricing`, reexport del motor
`app.catalog_engine.pricing`, spec 014) y `variant` ya están en el ámbito local de
`add_item_to_table` dos líneas antes del `199` (Constitución, Principio IV — no aplica
justificación porque no se añade nada)

**Storage**: PostgreSQL 16 schema-per-tenant en producción (sin cambio de esquema — el fix es de
comportamiento en tiempo de escritura, no de datos); SQLite en memoria vía
`app/characterization_tests/orders_fixtures.py` para los tests

**Testing**: `unittest` vía `python3 -m unittest` (mismo patrón que el resto de
`app/characterization_tests/`, sin `pytest.ini` en el repo) — el test existente
`test_add_item_to_table_a04_omite_validacion_de_seleccion_de_opciones`
(`test_orders_consolidation.py:65-83`) hoy **CONGELA el comportamiento defectuoso**; esta spec lo
modifica citando A-04 en el commit (Principio II) para que verifique el rechazo con `422` en vez
de la aceptación silenciosa, y añade el caso de paridad exacto que exige FR-006/FR-003

**Target Platform**: Linux server (`pos-backend`, API FastAPI en producción)

**Project Type**: corrección puntual dentro de un servicio backend único (no hay frontend
involucrado — la terminal del mesero ya consume el mismo endpoint de `add_item_to_table`; el único
cambio observable es que una selección hoy inválida empieza a devolver `422` en vez de `200`, sin
cambio de contrato de request/response)

**Performance Goals**: sin objetivo nuevo — la corrección añade la misma llamada a
`validate_option_selection` que ya paga hoy `create_order` para la variante equivalente; sin
consulta adicional más allá de la que ya hace ese camino de referencia

**Constraints**: no se toca `load_valid_options` ni `validate_option_selection`
(`app/catalog_engine/pricing.py`, spec 004, fuera de alcance) — Principio III, un módulo a la vez;
no se toca ningún otro caller de `load_valid_options` (`service.create_order`, ya correcto;
`kitchen.void_item` vía `repl_variant`, ya correcto); no se recalcula ningún `OrderItem`/orden/
factura ya generados con selección incompleta antes del cambio (FR-005)

**Scale/Scope**: 1 línea en 1 función de 1 fichero de producción (`consolidation.py:199`,
`add_item_to_table`); 1 test existente a modificar con cita de decisión
(`test_orders_consolidation.py:65-83`) + 3 tests nuevos (selección completa sin cambio, exceso de
`max_select`, paridad con `create_order` — FR-006); ningún fichero de `pos-heladeria` ni migración
de base de datos

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| Principio | Evaluación | Estado |
|---|---|---|
| **I. El Comportamiento Sigue Siendo Sagrado (por Defecto)** | El cambio de comportamiento está autorizado por escrito en `registro-de-anomalias.md`, entrada A-04 ("BUG HISTÓRICO CON DEPENDIENTES", tratamiento reforzado), más P4 de `entrevista-negocio.md` (jefe de cocina, 2026-08-16, merma real confirmada) — citado en `spec.md` §Autorización de negocio. Ningún otro comportamiento cambia por criterio técnico. | PASS |
| **II. Los Characterization Tests son el Árbitro** | `test_add_item_to_table_a04_omite_validacion_de_seleccion_de_opciones` (prefijo `"CONGELA comportamiento actual"` en su docstring, `test_orders_consolidation.py:65`) congela hoy el defecto — se modifica citando A-04 en el commit, como exige el propio Principio II. FR-006 exige además el caso de paridad exacto entre los dos caminos. Ningún otro test `CONGELA` de `pos-backend` se toca — en particular `test_rn_cat_33_a04_sin_pasar_variant_load_valid_options_no_valida_nada` (`test_catalog_line_pricing.py:206`) sigue vigente tal cual: documenta el mecanismo de `load_valid_options` sin `variant=`, no el caller, y esta delta no cambia `load_valid_options`. | PASS (modificación autorizada y citada) |
| **III. Estrangulamiento antes que Reescritura** | Un solo módulo en juego: `consolidation.add_item_to_table`, una línea. `load_valid_options`/`validate_option_selection` (spec 004) y los demás callers (`create_order`, `void_item`) quedan explícitamente sin tocar (Out of Scope de `spec.md`). No hay otra extracción/reescritura en curso que se solape. | PASS |
| **IV. Dependencias Nuevas Permitidas con Justificación** | No se añade ninguna dependencia ni ningún import — `variant` ya está en el ámbito local de `add_item_to_table` (`consolidation.py:196`), dos líneas antes del `199` que cambia. | PASS (no aplica) |
| **V. Ningún Cambio Retroactivo** | FR-005 lo exige explícitamente y `add_item_to_table` no tiene mecanismo de recálculo — el cambio solo afecta altas ejecutadas **después** del despliegue. Ningún `OrderItem`, orden ni factura ya emitidos se toca. | PASS |
| **VI. Todo en Español de Colombia** | Esta spec, plan y los artefactos que genera (research.md, data-model.md, contracts/, quickstart.md) se escriben en español de Colombia, igual que la spec de origen (009) y el resto de `pos-specs`. | PASS |

Sin violaciones. La tabla de Complexity Tracking al final de este documento queda vacía.

## Project Structure

### Documentation (this feature)

```text
specs/020-correccion-validacion-opciones-mesero/
├── plan.md              # Este fichero (/speckit-plan)
├── research.md          # Fase 0 (/speckit-plan) — por qué una línea basta y no reintroduce ee94f30
├── data-model.md        # Fase 1 (/speckit-plan)
├── quickstart.md        # Fase 1 (/speckit-plan)
├── contracts/
│   └── add-item-to-table-endpoint.md  # Fase 1 (/speckit-plan) — contrato HTTP (nuevo 422 documentado)
└── tasks.md              # Fase 2 (/speckit-tasks — NO se crea en este comando)
```

### Source Code (repositorio `../pos-backend`, sibling de `pos-specs`)

Esta spec vive en `pos-specs`, pero el código que describe está en el repositorio sibling
`pos-backend` (`../pos-backend` relativo a `pos-specs`, según la Constitución §Alcance). Rutas
listadas relativas a la raíz de `pos-backend`.

```text
app/
├── api/v1/orders/
│   ├── consolidation.py       # ÚNICO fichero de producción que cambia — add_item_to_table:199
│   │                            # pasa variant=variant a load_valid_options (FR-001/FR-002);
│   │                            # el resto de la función (routing de orden, expansión de combo,
│   │                            # deduct_order_items) SIN CAMBIOS
│   ├── service.py               # SIN CAMBIOS — create_order:102 ya pasa variant=variant, es la
│   │                             # referencia que esta corrección replica
│   ├── kitchen.py                # SIN CAMBIOS — void_item ya pasa variant=repl_variant, correcto
│   └── router.py                  # SIN CAMBIOS — mismo endpoint, mismo response_model; el único
│                                   # cambio observable es que una selección hoy inválida devuelve
│                                   # 422 en vez de 200
│
├── api/v1/catalog/line_pricing.py    # SIN CAMBIOS — fachada de app.catalog_engine (spec 014)
├── catalog_engine/pricing.py          # SIN CAMBIOS — load_valid_options/validate_option_selection,
│                                        # contrato ya fijado por la spec 004 (RN-CAT-33); esta
│                                        # delta solo cambia cómo la llama add_item_to_table
│
└── characterization_tests/
    ├── test_orders_consolidation.py   # Se MODIFICA el test T012 existente (cita A-04 en el
    │                                   # commit, Principio II): pasa de verificar la aceptación
    │                                   # silenciosa a verificar el rechazo con 422; se añade el
    │                                   # caso de paridad exacto entre add_item_to_table y
    │                                   # create_order que exige FR-006
    ├── test_orders_service.py         # SIN CAMBIOS — el test de contraste A-04 del lado de
    │                                   # create_order ya documenta el comportamiento correcto que
    │                                   # esta delta extiende al otro camino
    ├── test_catalog_line_pricing.py   # SIN CAMBIOS — test_rn_cat_33_a04_sin_pasar_variant... 
    │                                   # documenta el mecanismo de load_valid_options en sí
    │                                   # (spec 004), no el caller que esta delta corrige
    └── orders_fixtures.py             # SIN CAMBIOS — make_option_group/link_variant_group ya
                                        # existentes cubren lo que necesita el caso de paridad
```

**Structure Decision**: cambio de una sola línea en un fichero de producción ya existente
(`app/api/v1/orders/consolidation.py:199`), sin paquete ni módulo nuevo — no es una extracción
(Principio III no exige "estrangulamiento" para restaurar un argumento perdido en un merge dentro
de un módulo que la spec 017 ya caracterizó como propio). `variant` ya está en el ámbito local de
`add_item_to_table` (asignado en la línea 196, `get_or_404(db, ProductVariant, ...)`), así que el
fix no agrega ningún import ni dependencia nueva — verificado leyendo el cuerpo completo de la
función.

## Complexity Tracking

> **Fill ONLY if Constitution Check has violations that must be justified**

Sin violaciones — tabla vacía.
