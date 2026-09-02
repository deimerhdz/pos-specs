# Implementation Plan: Mejoras de usabilidad en el formulario de administración de promociones

**Branch**: `071-mejoras-formulario-promociones` | **Date**: 2026-09-02 | **Spec**: [spec.md](./spec.md)

**Input**: Feature specification from `/specs/071-mejoras-formulario-promociones/spec.md`

## Summary

Cuatro mejoras de usabilidad sobre el formulario de administración de promociones
(`pos-heladeria`), acotadas al componente `PromotionsPageComponent` y a un único endpoint de
`pos-backend`. Tres son correcciones de presentación/interacción sin impacto en datos ni en el
motor de cobro; la cuarta relaja un bloqueo de edición que la spec 063 impuso también sobre el
estado `Pausada` (documentado como A-69 en
[registro-de-anomalias.md](../000-reconocimiento/registro-de-anomalias.md), Principio II).

El reconocimiento de código (antes de escribir este plan) encontró que las cuatro historias se
resuelven sin ningún endpoint nuevo, sin cambio de esquema y sin ninguna dependencia nueva:

1. **Resumen de regla con nombre de producto (US1)** — `ruleSummaryText(rule)`
   ([promotions-page.component.ts:1043-1050](../../../pos-heladeria/src/app/modules/promotions/pages/promotions-page.component.ts#L1043-L1050))
   es la única función que produce el texto roto; es independiente de `ruleConditionPreview`, que
   la spec 066 ya corrigió para la pantalla de revisión. Pasa a recibir el índice de la regla,
   reutiliza `setDescriptor()` de `promotion-condition.util.ts` (ya exportado por la spec 066)
   para el orden/tope de nombres, y arma su propia oración
   ([contracts/resumen-de-regla.md](./contracts/resumen-de-regla.md)).
2. **Selección por búsqueda, no por catálogo completo (US2)** — `visibleVariantsForRule`
   ([línea 867](../../../pos-heladeria/src/app/modules/promotions/pages/promotions-page.component.ts#L867))
   se separa en `selectedVariantsForRule` (ya existía, sin cambios: es el listado del conjunto) y
   `searchResultsForRule` (antes `visibleVariantsForRule`, gana la guarda "sin categoría y sin
   texto → vacío"). Ambos operan sobre `catalogVariants()`, ya cargado en memoria — cero llamadas
   nuevas a la API ([contracts/busqueda-y-seleccion.md](./contracts/busqueda-y-seleccion.md)).
3. **Regla nueva al principio (US3)** — `addRule()` cambia `push` por `unshift` en `form.rules` y
   en `ruleFilters` (arreglo paralelo por índice), y expande la posición 0 en vez de la última.
4. **Editar el conjunto de una promoción `Pausada` (US4)** — un único condicional en
   `service.update_shape`
   ([service.py:760](../../../pos-backend/app/api/v1/promotions/service.py#L760)): de
   `!= "draft"` a `not in ("draft", "paused")`. Todas las validaciones de guardado que la spec
   necesita (solape FR-014/FR-014a, conjuntos disjuntos FR-001a, precio de paquete que sí
   descuente FR-016, mínimo de una regla) **ya corren** dentro de ese método sin condicionar por
   estado más allá de excluir la propia promoción. En el cliente, `canEditShape()` se separa en
   `canEditRuleSet()` (agregar/quitar reglas y conjunto — habilitado en `Pausada`) y
   `canEditRuleTypeValue(rule)` (tipo/valor/cantidad mínima — sigue bloqueado en `Pausada` para
   una regla que ya existía, vía el nuevo campo cliente-only `PromotionRuleForm.isExisting`)
   ([contracts/edicion-en-pausada.md](./contracts/edicion-en-pausada.md)).

Detalle de las decisiones D1-D5 en [research.md](./research.md).

## Technical Context

**Language/Version**: Python 3.14 (`pos-backend`) · TypeScript 5.x / Angular 20 standalone,
signals (`pos-heladeria`) — sin cambio de versión en ninguno de los dos repos.

**Primary Dependencies**: FastAPI + Pydantic v2 + SQLAlchemy 2.0 (backend, sin cambio);
Angular standalone con `@if`/`@for`, Tailwind (frontend, sin cambio). **Ninguna dependencia
nueva** (Principio IX: nada que justificar).

**Storage**: PostgreSQL 16, schema-per-tenant. **Sin cambio de esquema y sin migración**
([data-model.md](./data-model.md)): todo lo que cambia es a qué estado de `Promotion` se le
permite reemplazar su lista de `PromotionRule`, no la forma de esa lista.

**Testing**: `python -m unittest discover -s app/characterization_tests -p 'test_*.py' -v`
(backend, sin `pytest`/`conftest.py`, fixtures SQLite en memoria vía `cart_fixtures.py`) +
Karma/Jasmine (`*.spec.ts`, frontend).

**Target Platform**: API en Linux; frontend web responsive — el formulario de administración se
usa en escritorio, sin restricción nueva de dispositivo.

**Project Type**: Web application, dos repositorios independientes ya en producción
(`pos-backend`, `pos-heladeria`).

**Performance Goals**: SC-002 (encontrar y agregar un producto en <10s con >100 variantes) se
cumple con filtrado en memoria sobre `catalogVariants()`, ya cargado una vez por `ngOnInit`
(research.md D2) — no hay ninguna llamada de red nueva que perfilar.

**Constraints**: `Activa` sigue bloqueando reglas por completo (FR-013, sin cambio de spec 063);
tipo/valor/cantidad mínima de una regla que ya existía siguen bloqueados en `Pausada` (FR-015);
ningún cambio afecta el motor de cobro (`evaluate_variant_sets`) ni promociones `Activa`
(`Pausada` nunca aplica descuento — `active_variant_set_rules` solo lee `status == "active"`);
texto de UI en español de Colombia (Principio XIII).

**Scale/Scope**: 2 repositorios, 0 endpoints nuevos, 0 migraciones. Backend: 1 archivo de lógica
(`promotions/service.py`, una condición), 1 de router (docstring/summary), tests nuevos/editados
en `test_promotions_rules_admin.py`. Frontend: 1 componente
(`promotions-page.component.ts` + su plantilla inline), 1 interfaz (`promotion.interface.ts`,
campo `isExisting`), reutiliza sin modificar `promotion-condition.util.ts`
(`promotion.service.ts` no cambia: `toRules()` ya ignora campos no declarados en el payload).

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

Evaluado contra la [Constitución v3.0.0](../../.specify/memory/constitution.md).

| Principio | Estado | Evidencia |
|---|---|---|
| I. Las nuevas funcionalidades nacen de un spec | ✅ | [spec.md](./spec.md) aprobada, Clarifications cerradas el 2026-09-02. |
| II. El comportamiento existente sigue protegido | ✅ | El único cambio de comportamiento (US4/FR-013 a FR-018) queda registrado como **A-69** en [registro-de-anomalias.md](../000-reconocimiento/registro-de-anomalias.md), con quién/cuándo/qué cambia, antes de esta implementación. Las otras tres historias son correcciones de presentación/interacción, no decisiones de negocio (no cambian ninguna regla ya documentada). |
| III. Los characterization tests protegen el comportamiento heredado | ✅ | `test_ca2_cambiar_reglas_de_una_activa_bloquea` (el único test que toca `update_shape` con un estado no-`draft`) prueba `Activa`, que no cambia, y no lleva el prefijo `"CONGELA comportamiento actual:"` — verificado por `grep` sobre los cuatro ficheros de `characterization_tests/` que tocan promociones (research.md D5). Nada que negociar. |
| IV. Los nuevos specs pueden introducir nuevo comportamiento | ✅ | El criterio de éxito es la conformidad con esta spec (SC-001 a SC-005), no la equivalencia con producción. |
| V. Nuevas funcionalidades antes que refactorizaciones oportunistas | ✅ | Cada cambio se ata a un FR concreto. `visibleVariantsForRule` se **renombra y se le agrega una guarda**, no se reescribe; `setDescriptor()` se **reutiliza**, no se duplica. |
| VI. Evolución incremental | ✅ | Cuatro historias entregables por separado (tabla de fases abajo), sin mezclar refactorización, arquitectura ni migración de datos con el cambio de comportamiento de US4. |
| VII. Compatibilidad con datos históricos | ✅ | Ninguna venta, factura ni pedido se toca. `Pausada` nunca aplicó descuento y sigue sin hacerlo — editar su conjunto no reescribe ningún cobro pasado ni en curso. |
| VIII. Evolución del modelo de datos | ✅ N/A | Cero cambios de esquema, cero migraciones ([data-model.md](./data-model.md)). El único campo nuevo (`isExisting`) es de un formulario en memoria del navegador, no de una tabla. |
| IX. Dependencias nuevas justificadas | ✅ N/A | Ninguna dependencia nueva en ningún repositorio. |
| X. Verificación obligatoria | ✅ | [quickstart.md](./quickstart.md) enumera los escenarios ejecutables por historia; los tres contratos fijan los textos y tablas de casos que los tests deben afirmar literalmente. |
| XI. Decisiones de negocio frente a técnicas | ✅ | La única decisión de negocio (relajar FR-018 en `Pausada`) está aislada en A-69; las técnicas (dos permisos en vez de uno, dónde vive `isExisting`, cómo separar los dos listados) viven en research.md D1-D4. |
| XII. Trazabilidad | ✅ | Necesidad (4 defectos reportados con captura) → spec 071 → Clarifications → A-69 → este plan → contratos → tasks.md → tests → quickstart. |
| XIII. Todo en español de Colombia | ✅ | Todos los textos de UI nuevos (resumen de regla, mensaje de error de `update_shape`) y los nombres de test nuevos, en español de Colombia. |

### Re-evaluación post-diseño (Phase 1)

Repasada tras generar data-model, los tres contratos y quickstart. **Sin violaciones nuevas.**

- **VIII confirmado, no solo declarado**: el diseño de `isExisting` (research.md D3) cerró en
  cero campos de backend/esquema — quedó en el formulario del cliente, exactamente como se
  proyectaba antes de diseñar el detalle.
- **III confirmado con más precisión**: el diseño identificó el `test_ca2` exacto que hay que
  dejar intacto (research.md D5), no solo "no hay tests protegidos" en general.

Ningún gate queda en rojo.

## Project Structure

### Documentation (this feature)

```text
specs/071-mejoras-formulario-promociones/
├── plan.md              # Este archivo
├── research.md          # Decisiones D1-D5
├── data-model.md         # Cero cambios de esquema; PromotionRuleForm.isExisting
├── contracts/
│   ├── resumen-de-regla.md          # FR-001 a FR-005
│   ├── busqueda-y-seleccion.md      # FR-006 a FR-011
│   └── edicion-en-pausada.md        # FR-012 a FR-018
├── quickstart.md        # Validación ejecutable
└── tasks.md              # Salida de /speckit-tasks (no de este comando)
```

### Source Code (repository root)

```text
../pos-backend/
├── app/api/v1/promotions/service.py     # update_shape: condición de estado (1 línea)
├── app/api/v1/promotions/router.py      # summary del endpoint /shape
└── app/characterization_tests/
    └── test_promotions_rules_admin.py   # tests nuevos: Pausada permite shape; Activa sigue bloqueada (sin tocar)

../pos-heladeria/
└── src/app/modules/promotions/
    ├── interfaces/promotion.interface.ts       # PromotionRuleForm.isExisting (nuevo campo)
    ├── pages/promotions-page.component.ts      # ruleSummaryText, searchResultsForRule,
    │                                            #   selectedVariantsForRule (reutilizado),
    │                                            #   addRule (unshift), canEditRuleSet,
    │                                            #   canEditRuleTypeValue, isPaused, save()
    └── pages/promotions-page.component.spec.ts # tests nuevos por historia
```

**Structure Decision**: se conserva la separación de los dos repositorios en producción, sin
crear módulos, servicios ni capas nuevas. Todo el trabajo de frontend vive en el componente que ya
existe; `promotion-condition.util.ts` se **reutiliza tal cual** (ya expone `setDescriptor`) sin
ganar ningún archivo nuevo, a diferencia de la spec 066 que sí tuvo que crearlo.

## Fases de entrega (Principio VI)

| Fase | Historia | Alcance | Se puede desplegar sola |
|---|---|---|---|
| **0** | — | Confirmar A-69 registrada (ya lo está, ver Constitution Check). | — |
| **1** | US1 (P1) | `ruleSummaryText(ruleIndex)` con nombres, reutilizando `setDescriptor`. | Sí. |
| **2** | US2 (P2) | `searchResultsForRule` + plantilla con dos listados separados. | Sí. |
| **3** | US3 (P3) | `addRule()` inserta en la posición 0. | Sí. |
| **4** | US4 (P1) | `update_shape` acepta `paused`; `canEditRuleSet`/`canEditRuleTypeValue`/`isExisting`/`isPaused` en el cliente; `save()` llama `updateShape` también en `paused`. | Sí, es independiente de las tres anteriores. |

## Complexity Tracking

> Sin violaciones del Constitution Check que justificar — tabla omitida.
