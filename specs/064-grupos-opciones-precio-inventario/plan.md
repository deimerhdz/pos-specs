# Implementation Plan: Tipo de precio e inventario condicional en grupos de opciones

**Branch**: `064-grupos-opciones-precio-inventario` | **Date**: 2026-09-01 | **Spec**: [spec.md](./spec.md)

**Input**: Feature specification from `/specs/064-grupos-opciones-precio-inventario/spec.md`

## Summary

Esta spec agrega un tipo explícito (`pricing_type`: "Incluido"/"Con recargo") a `OptionGroup`,
para que un sabor incluido en el precio del producto y un topping con recargo dejen de depender de
la convención "dejar `extra_price` en $0 a mano". Extiende el switch `Product.tracks_inventory`
(spec 027) para que también gobierne los campos de insumo/cantidad de consumo de los grupos de
opciones de un producto — hallazgo clave de la exploración de código: hoy esos campos están
enredados con la propia posibilidad de ofrecer un grupo de opciones en absoluto (`product-form.component.ts:275-413`),
así que esta spec **restructura** ese bloque, no solo lo oculta más. Corrige la discrepancia ya
documentada (anomalía A-32, `RN-CAT-39`) entre `grupos_que_descuentan` y `group_discounts`
unificándolas hacia el criterio más estricto, ya tratado como canónico en la migración de spec 027.
Y extiende el gating por plan (spec 033/062) al switch de inventario y a los campos de insumo de
una opción — pero a nivel de **campo**, no de ruta completa, porque ambos endpoints deben seguir
funcionando para un tenant sin el módulo Inventario (crear productos, crear toppings con precio).

Cero endpoints nuevos, cero entidades nuevas: 1 columna nueva (`option_groups.pricing_type`), 3
funciones de validación de servicio nuevas, 1 función de backend extraída para reutilización
(`ensure_module_access`), 1 función de negocio modificada deliberadamente (`grupos_que_descuentan`),
y una restructuración de un bloque de plantilla en el frontend.

## Technical Context

**Language/Version**: Backend — Python 3.14.4 (venv `pos-backend/env`). Frontend — TypeScript
5.9.2 (Angular 21.1.x, standalone components + signals), Node 24.16.0.

**Primary Dependencies**: Ninguna dependencia nueva en ningún repositorio.
- Backend: SQLAlchemy + Alembic (ya en uso), `app.core.plan_limits` (spec 033, se le extrae una
  función pura nueva sin cambiar su API pública), `app.catalog_engine.pricing`/`consumption` (spec
  014, se modifica una función interna sin cambiar la superficie pública re-exportada).
- Frontend: reutiliza `PlanSummaryService`/`ModuleAccess` (`modules/plan/`, spec 033/062),
  `ConfirmService` (ya usado por `toggleTracksInventory()`, spec 027), `ReactiveFormsModule` (ya en
  uso en `option-group-form.component.ts`/`option-form.component.ts`).

**Storage**: PostgreSQL 16, schema-per-tenant. **1 migración nueva**: `option_groups` gana la
columna `pricing_type` (`String(20)` + `CheckConstraint`), con backfill por `UPDATE ... CASE WHEN
EXISTS` (data-model.md). Ninguna otra tabla cambia de forma.

**Testing**: Backend — `unittest` vía `python -m unittest discover -s app/characterization_tests -p
'test_*.py'`, mismo patrón que `test_catalog_line_pricing.py`/`test_catalog_consumption_plan.py`/
`test_plan_module_access.py` (todos existentes, todos tocados por esta spec). Frontend — Vitest vía
`ng test`, specs colocados; `product-form.component.spec.ts` (existente) se extiende, y se crean
`option-group-form.component.spec.ts`/`option-form.component.spec.ts` (hoy no existen para estos
dos componentes).

**Target Platform**: Linux server (API `pos-backend`) + navegador (SPA `pos-heladeria`).

**Project Type**: Web application (backend FastAPI + frontend Angular, repos sibling de
`pos-specs`, estructura ya establecida por specs 002-062).

**Performance Goals**: Ninguno nuevo. La validación cruzada de `pricing_type` es una lectura de una
fila ya cargada (`OptionGroup` vía `get_or_404`) sin `JOIN` ni consulta adicional. El gating de plan
por campo reutiliza exactamente el mismo costo que `enforce_plan_limit` ya paga hoy en
`POST /products` (un `SELECT` de `Plan` por PK).

**Constraints**:
- `POST`/`PATCH /products` y `POST /option-groups/{id}/options`/`PATCH /options/{id}` **no pueden**
  gatearse a nivel de router completo (a diferencia de `unit_measures`/`reports` en spec 062) —
  deben seguir aceptando productos y opciones que no activen ningún campo de inventario, sin
  importar el plan del tenant (research.md Decisión 4).
- El bloque "Sabores a elegir" de `product-form.component.ts` (líneas 302-408) debe partirse en dos
  niveles de visibilidad (selector de grupo + min/max siempre visibles; cantidad de consumo y
  detalle de insumo solo con inventario activo) — no puede seguir siendo un único `@if`, porque hoy
  esconde también la posibilidad de ofrecer el grupo en absoluto (research.md Decisión 5).
- Modificar `grupos_que_descuentan` toca una función con characterization tests que fijan
  explícitamente el comportamiento discrepante de A-32 (`test_catalog_line_pricing.py`) — el cambio
  exige actualización explícita de esos tests citando esta spec como autorización (Principio III de
  la Constitución), no un ajuste silencioso.
- Ningún dato existente se pierde ni se recalcula: la migración de `pricing_type` es puramente
  clasificatoria (SC-002); retirar el acceso a Inventario nunca borra `tracks_inventory` ni
  `inventory_item_id`/`item_quantity` ya guardados (FR-013).

**Scale/Scope**: 1 tabla modificada (`option_groups`, +1 columna), 0 tablas nuevas. Backend: 1
migración nueva, 1 función extraída (`plan_limits.py`), 1 función modificada
(`catalog_engine/pricing.py`), 2 endpoints ganan validación cruzada + `tenant` (`catalog/router.py`:
`add_option`, `update_option`), 2 endpoints ganan validación de plan (`products/service.py`:
`create_product`, `update_product`; `update_product`'s router gana `tenant`), 1 endpoint gana efecto
lateral (`update_option_group`). Frontend: 1 componente restructurado (`product-form.component.ts`),
2 componentes ganan gating por plan (`option-group-form.component.ts`, `option-form.component.ts`,
más su página contenedora `option-groups-page.component.ts` para inyectar `PlanSummaryService`), 1
interfaz TypeScript extendida (`product.interface.ts`: `OptionGroup.pricing_type`).

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| Principio | Evaluación | Estado |
|---|---|---|
| **I. Las Nuevas Funcionalidades Nacen de un Spec** | `spec.md` existe, aprobado, 6 historias priorizadas, 15 FRs, sesión de Clarifications (2026-09-01) que resolvió las 4 decisiones de diseño antes de este plan. | PASS |
| **II. El Comportamiento Existente Sigue Protegido** | Tres cambios de comportamiento, los tres ya documentados y autorizados en `spec.md`: (1) unificar el criterio de A-32 (FR-009, ver Principio III); (2) el switch de inventario y los campos de insumo ahora exigen el módulo Inventario en el plan (FR-011, gap que antes no existía — spec 027 no mencionaba plan alguno); (3) "Sabores a elegir" deja de ocultarse completo sin inventario (FR-006, corrige una sobre-restricción no buscada de spec 027, no un comportamiento deliberado documentado en esa spec). Los tres requieren entrada en `registro-de-anomalias.md` (research.md Decisión 7 para A-32; los otros dos son comportamiento nuevo de un spec funcional, no corrección de una regla heredada, mismo razonamiento que spec 033/062 aplicaron a sus propios gatings nuevos — no exigen entrada retroactiva). | PASS (con tarea de trazabilidad pendiente en `tasks.md`, ver Principio III) |
| **III. Los Characterization Tests Protegen el Comportamiento Heredado** | `test_catalog_line_pricing.py` y `test_catalog_consumption_plan.py::EnsureLinesConsumeInventoryTests` fijan hoy el comportamiento discrepante de A-32 — se actualizan explícitamente (research.md Decisión 3), citando esta spec y la entrada nueva en `registro-de-anomalias.md` (Decisión 7) como autorización, no un ajuste para "hacer pasar" el cambio. `product-form.component.spec.ts` fija hoy que "Sabores a elegir" desaparece completo sin inventario (research.md Decisión 5) — se actualiza para reflejar la restructuración, con la misma trazabilidad. | PASS |
| **IV. Los Nuevos Specs Pueden Introducir Nuevo Comportamiento** | El criterio de éxito es conformidad con `spec.md` (SC-001 a SC-006), no equivalencia total con el pasado — coherente con que esta spec corrige explícitamente dos comportamientos previos (A-32, el ocultamiento excesivo de "Sabores a elegir"). | PASS |
| **V. Nuevas Funcionalidades Antes que Refactorizaciones Oportunistas** | Extraer `ensure_module_access` de `require_module_access` (research.md Decisión 4) no es una refactorización ajena — es el mecanismo mínimo para reutilizar la misma comprobación de plan en un gating de campo, sin duplicar la lógica de `ensure_plan_not_expired` + lookup de `Plan`. No se toca ningún otro consumidor de `require_module_access` (spec 033/062 siguen funcionando idénticos, misma firma). | PASS |
| **VI. Evolución Incremental** | Las seis historias son independientemente verificables: US1/US2 (tipo de grupo) no dependen de US3 (cascada del switch) ni de US4 (unificar A-32) ni de US5 (gating por plan) para funcionar ni para probarse — cada una toca archivos disjuntos salvo `catalog/router.py`, donde las validaciones de US1/US2 y US5 conviven en los mismos dos endpoints (`add_option`/`update_option`) sin interferir (una valida `pricing_type`, la otra valida acceso a módulo). | PASS |
| **VII. Compatibilidad con Datos Históricos** | Ninguna `Sale`/`Payment`/`Invoice` se toca. La migración de `pricing_type` no recalcula ningún precio de línea ya vendido — es una etiqueta nueva sobre `OptionGroup`, sin relación con ventas pasadas. Vender hoy con un grupo migrado produce el mismo precio y el mismo consumo que antes de la migración (SC-002, verificado en quickstart.md US6). | PASS |
| **VIII. Evolución del Modelo de Datos** | `data-model.md` especifica la columna nueva (`option_groups.pricing_type`), su tipo, nulabilidad, default (asimétrico ORM vs. schema, research.md Decisión 1), compatibilidad con datos existentes (backfill por `EXISTS`, Decisión 6) y estrategia de rollback (`downgrade` con `drop_constraint`+`drop_column`, reversible sin pérdida de otros datos — se pierde únicamente la clasificación, recalculable de nuevo desde `extra_price` si se necesita reaplicar). | PASS |
| **IX. Dependencias Nuevas Permitidas con Justificación** | No aplica — cero dependencias nuevas en `requirements.txt`/`package.json`. | N/A |
| **X. Verificación Obligatoria** | Cada historia de usuario tiene su "Independent Test" en `spec.md`; `quickstart.md` los traduce a pasos ejecutables por historia, incluyendo qué characterization tests existentes se extienden (no solo cuáles se crean) — ver Principio III para el caso específico de A-32. | PASS |
| **XI. Decisiones de Negocio Frente a Decisiones Técnicas** | Las cuatro decisiones de negocio (tipo de grupo explícito, cascada del switch, unificar A-32, gating por plan) ya quedaron registradas en `spec.md` (Clarifications) antes de este plan. Las decisiones de este plan son técnicas: nombre/tipo de la columna (Decisión 1), dónde vive la validación cruzada (Decisión 2), hacia cuál de los dos criterios discrepantes unificar — la de `group_discounts`, ya tratada como canónica en código existente (Decisión 3), y la forma del gating de plan — por campo, no por ruta (Decisión 4). Ninguna decisión técnica cambia qué ve o puede hacer el usuario más allá de lo ya definido en el spec. | PASS |
| **XII. Trazabilidad** | Cadena completa: `spec.md` (Necesidad+Spec+Clarifications) → este `plan.md`/`research.md`/`data-model.md`/`contracts/` (Decisión técnica) → entrada nueva en `registro-de-anomalias.md` para A-32 (Decisión 7, tarea explícita de `tasks.md`) → `tasks.md` (Fase 2, no generada por este comando) → implementación → tests nuevos/actualizados por historia → `quickstart.md` (Verificación). | PASS |
| **XIII. Todo en Español de Colombia** | Este plan y todos sus artefactos se escriben en español de Colombia, igual que `spec.md`. | PASS |

Sin violaciones que no puedan resolverse dentro del propio spec/plan. La tabla de Complexity
Tracking al final de este documento queda vacía.

**Re-chequeo post-diseño (Fase 1)**: `research.md`, `data-model.md` y `contracts/` no introdujeron
ninguna entidad, dependencia ni decisión que contradiga la tabla anterior. Un punto merece mención
explícita: la exploración de código reveló que la restructuración de "Sabores a elegir" (Principio
II, fila del medio) es más profunda de lo que `spec.md` describía literalmente al nivel de detalle
de implementación — pero el comportamiento resultante (selector de grupo y precio siempre
editables, insumo/cantidad condicionados) es exactamente el que `spec.md` FR-006 ya exigía; no hay
contradicción entre el spec y el diseño, solo un hallazgo de que el código actual estaba más lejos
de ese objetivo de lo que una lectura superficial del spec sugería. Gates siguen en PASS.

## Project Structure

### Documentation (this feature)

```text
specs/064-grupos-opciones-precio-inventario/
├── plan.md              # Este fichero (/speckit-plan)
├── research.md          # Fase 0 — decisiones técnicas y alternativas descartadas
├── data-model.md         # Fase 1 — columna nueva, validaciones cruzadas, migración
├── quickstart.md         # Fase 1 — validación ejecutable por historia de usuario
├── contracts/            # Fase 1 — contratos de los endpoints tocados
│   ├── option-group-pricing-type.md
│   └── inventory-field-plan-gating.md
└── tasks.md              # Fase 2 (/speckit-tasks — NO se crea en este comando)
```

### Source Code (repositorios sibling de `pos-specs`)

Esta spec vive en `pos-specs`, pero el código que describe está en `../pos-backend` y
`../pos-heladeria` (Constitución §Alcance). Rutas relativas a la raíz de cada repo.

```text
# pos-backend
app/models/
└── option_group.py                    # MODIFICADO — nueva columna `pricing_type: Mapped[str]`
                                          + `CheckConstraint("pricing_type IN ('incluido',
                                          'con_recargo')", name="ck_option_group_pricing_type")`
                                          (data-model.md)

alembic/versions/
└── <hex>_option_groups_pricing_type.py # NUEVO — `down_revision = '94b7e35f5e5e'` (head actual).
                                          `op.add_column` + `op.create_check_constraint` +
                                          backfill `UPDATE ... CASE WHEN EXISTS` (research.md
                                          Decisión 6), mismo patrón `_has_table` +
                                          `@for_each_tenant_schema` que
                                          `b8c9d0e1f2a3_option_group_active.py` y
                                          `e3f4a5b6c7d8_products_tracks_inventory.py`

app/api/v1/catalog/
├── schemas.py                          # MODIFICADO — `OptionGroupCreate.pricing_type: str`
                                          (requerido, sin default, líneas ~150-155),
                                          `OptionGroupUpdate.pricing_type: str | None`,
                                          `OptionGroupResponse.pricing_type: str`
                                          (contracts/option-group-pricing-type.md)
└── router.py                           # MODIFICADO — 4 endpoints tocados:
                                          `create_option_group` (línea 281): persiste
                                          `pricing_type` del body;
                                          `update_option_group` (línea 301): al pasar de
                                          "con_recargo" a "incluido", `UPDATE options SET
                                          extra_price=0 WHERE option_group_id=:id` (FR-004);
                                          `add_option` (línea 347): gana `tenant: Tenant =
                                          Depends(get_tenant)`, valida `pricing_type` del grupo
                                          contra `extra_price` del body (422 si conflicto), y
                                          llama `ensure_module_access(db, tenant, "inventario")`
                                          si `inventory_item_id is not None or item_quantity > 0`
                                          (FR-002, FR-011);
                                          `update_option` (línea 380): mismos dos gates que
                                          `add_option`, evaluados sobre el resultado final tras
                                          aplicar el `PATCH`

app/core/
└── plan_limits.py                      # MODIFICADO — se extrae `ensure_module_access(db,
                                          tenant, module_key) -> None` (lógica interna de
                                          `require_module_access`, sin cambiar su firma ni
                                          comportamiento como dependencia FastAPI —
                                          research.md Decisión 4)

app/api/v1/products/
├── service.py                          # MODIFICADO — `create_product` (línea 55) y
                                          `update_product` (línea 87) llaman
                                          `ensure_module_access(db, tenant, "inventario")`
                                          cuando `data.tracks_inventory is True` (y, en
                                          `update_product`, distinto del valor actual)
                                          (contracts/inventory-field-plan-gating.md)
└── router.py                           # MODIFICADO — `update_product` (línea 104) gana
                                          `tenant: Tenant = Depends(get_tenant)` (hoy no lo
                                          recibe; `create_product` ya lo tiene, línea 79)

app/catalog_engine/
└── pricing.py                          # MODIFICADO deliberadamente — `grupos_que_descuentan`
                                          (línea 62) gana las mismas dos condiciones que
                                          `group_discounts` (`Option.active.is_(True)`,
                                          `Option.inventory_item_id.is_not(None)`), corrigiendo
                                          A-32/RN-CAT-39 (research.md Decisión 3)

app/characterization_tests/
├── test_catalog_line_pricing.py        # MODIFICADO deliberadamente — el/los test(s) que fijan
                                          el criterio de una sola condición de
                                          `grupos_que_descuentan` se actualizan citando esta
                                          spec como autorización (Principio III)
├── test_catalog_consumption_plan.py    # SIN CAMBIO en `group_discounts` (ya es el criterio
                                          canónico) — puede ganar un caso nuevo que confirme que
                                          ambos criterios ahora coinciden
├── test_plan_module_access.py          # SIN CAMBIO de comportamiento — puede ganar un caso
                                          nuevo que invoque `ensure_module_access` directamente
                                          (misma cobertura genérica que ya da a
                                          `require_module_access`)
└── test_products_service.py            # MODIFICADO/EXTENDIDO — casos nuevos de
                                          `create_product`/`update_product` con
                                          `tracks_inventory=true` sin módulo Inventario → 403

specs/000-reconocimiento/
└── registro-de-anomalias.md            # MODIFICADO — entrada nueva marcando A-32 como
                                          resuelta por spec 064 (research.md Decisión 7),
                                          citando quién/cuándo/qué cambia/por qué

# pos-heladeria
src/app/modules/products/
├── interfaces/
│   └── product.interface.ts            # MODIFICADO — `OptionGroup.pricing_type: 'incluido' |
                                          'con_recargo'` (líneas ~157-164), y sus contrapartes
                                          en `OptionGroupForm`/`OptionGroupCreatePayload`/
                                          `OptionGroupUpdatePayload` (líneas 167-186)
├── pages/
│   ├── product-form.component.ts       # MODIFICADO — restructura "Sabores a elegir" (líneas
                                          302-408): selector de grupo + min/max fuera del `@if`
                                          de inventario; input "descuenta" + resumen/detalle de
                                          insumo permanecen condicionados a una señal nueva
                                          `sectionsEnabled()` (`draft().tracks_inventory &&
                                          inventarioIncluido()`); el switch "Maneja inventario"
                                          (línea 181) se deshabilita (no se oculta) cuando
                                          `!inventarioIncluido()`; inyecta `PlanSummaryService`
                                          (research.md Decisión 5)
│   └── product-form.component.spec.ts  # MODIFICADO — el test "la sección de insumos aparece
                                          deshabilitada mientras el switch está apagado" (línea
                                          104) se actualiza: el selector de grupo ya no
                                          desaparece, solo el input de cantidad; se agregan casos
                                          de gating por plan (switch deshabilitado sin módulo)
└── services/
    └── product.service.ts              # SIN CAMBIO DE FORMA — `toVariantSavePayload`/
                                          `saveProduct` no transportan `pricing_type` (es del
                                          `OptionGroup`, no de la asignación variante↔grupo)

src/app/modules/option-groups/
├── components/
│   ├── option-group-form.component.ts  # MODIFICADO — el `FormGroup` gana el control
                                          `pricing_type` (requerido, líneas 82-86); nuevo spec
                                          `option-group-form.component.spec.ts` (hoy no existe)
│   ├── option-form.component.ts        # MODIFICADO — el campo "Precio extra" (líneas 47-51) se
                                          deshabilita/oculta cuando el grupo padre es "incluido";
                                          los campos "Insumo que consume"/"Cantidad consumida"
                                          (líneas 53-73) se ocultan cuando `!inventarioIncluido()`
                                          (gating por plan, no por producto — research.md
                                          Decisión 5); recibe el `pricing_type` del grupo padre
                                          como `@Input()` adicional; nuevo spec
                                          `option-form.component.spec.ts` (hoy no existe)
│   ├── option-group-form.component.spec.ts  # NUEVO
│   └── option-form.component.spec.ts        # NUEVO
├── pages/
│   └── option-groups-page.component.ts # MODIFICADO — inyecta `PlanSummaryService` para pasar
                                          `inventarioIncluido()` a `option-form.component.ts`
└── services/
    └── option-group.service.ts         # MODIFICADO — `createGroup`/`updateGroup` incluyen
                                          `pricing_type` en el payload (líneas 66-82)
```

**Structure Decision**: Web application con dos repositorios (backend FastAPI + frontend Angular),
estructura ya establecida por specs previas (002 en adelante) — esta spec no agrega ningún
directorio ni módulo nuevo, modifica archivos existentes y crea 2 archivos de test frontend nuevos
más 1 migración backend nueva.

## Complexity Tracking

*Sin violaciones que justificar — tabla vacía.*
