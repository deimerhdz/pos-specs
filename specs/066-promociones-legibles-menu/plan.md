# Implementation Plan: Promociones legibles y precios reales en el menú QR

**Branch**: `066-promociones-legibles-menu` | **Date**: 2026-09-01 | **Spec**: [spec.md](./spec.md)

**Input**: Feature specification from `/specs/066-promociones-legibles-menu/spec.md`

## Summary

Tres defectos de **superficie de consumo** sobre el modelo de promociones que la spec
[063](../063-promociones-por-variante/spec.md) dejó en producción. No se toca el modelo de datos,
ni el motor de cálculo (`evaluate_variant_sets`), ni la administración de reglas: cambia qué ve
el comensal.

El enfoque técnico se apoya en un hallazgo del reconocimiento de código: **el backend ya es la
fuente única del texto de condición** (`variant_set_condition_text`) y **`ProductSelectComponent`
ya está compartido** entre el menú QR del comensal y la terminal del cajero. Eso permite resolver
las cuatro historias con tres movimientos:

1. **Backend, texto por nombres** — `variant_set_condition_text(rule)` pasa a
   `variant_set_condition_text(rule, names)`: recibe el mapa `variant_id → nombre utilizable` y
   construye el descriptor de FR-002 (deduplicar, ordenar sin tildes ni mayúsculas, resumir a
   tres). El parámetro es **obligatorio**, no opcional: los dos únicos call sites
   ([menu/router.py:206](../../../pos-backend/app/api/v1/menu/router.py#L206) y
   [promotions/service.py:685](../../../pos-backend/app/api/v1/promotions/service.py#L685)) quedan
   forzados a suministrarlo, y un call site olvidado falla en carga en vez de degradar en silencio
   al texto por conteo y romper SC-005.
2. **Backend, información de promoción por variante** — `MenuVariantResponse` gana un bloque
   aditivo `promotion` con la condición completa, la condición corta, el equivalente por unidad ya
   redondeado y ya renderizado como texto, y su marca de aproximado. Es lo que hace verificable
   SC-005 y lo que evita duplicar el redondeo de FR-009 en Python y en TypeScript.
   `discounted_price` conserva su significado y **extiende** su cobertura a `package_price` con
   `min_qty == 1` (FR-010, el defecto de importe).
3. **Frontend, pintar lo que llega** — `ProductSelectComponent` (compartido) pinta el bloque
   `promotion` por presentación: el comensal y el cajero ven el mismo texto sin ninguna rama por
   superficie. La tarjeta del menú QR deriva la insignia genérica de `variants.some(v =>
   v.promotion)`. El único cálculo local que queda es la vista previa del formulario de
   administración, porque describe variantes que todavía no se han guardado (FR-018).

**Riesgo principal identificado y acotado**: `MenuService` (la terminal) hoy **descarta**
`discounted_price` al mapear `GET /menu`. Se mantiene descartándolo y se mapea **solo**
`promotion`, para que la terminal gane la condición de FR-016 sin que ningún importe suyo cambie
(FR-017).

## Technical Context

**Language/Version**: Python 3.14 (`pos-backend`) · TypeScript 5.x / Angular 20 standalone,
signals (`pos-heladeria`)

**Primary Dependencies**: FastAPI + Pydantic v2 + SQLAlchemy 2.0 (backend); Angular con
componentes standalone, `@if`/`@for`, Tailwind (frontend). **Ninguna dependencia nueva** — el
único módulo que se suma es `unicodedata` de la biblioteca estándar, para normalizar tildes al
ordenar (Principio IX: no hay que justificar nada porque no se añade nada).

**Storage**: PostgreSQL 16, schema-per-tenant. **Sin cambios de esquema y sin migración**
(FR-019): todo lo que esta feature muestra es información derivada, calculada al momento de
responder.

**Testing**: `python -m unittest discover -s app/characterization_tests -p 'test_*.py' -v`
(sin pytest, sin `conftest.py`; fixtures sobre SQLite en memoria en `cart_fixtures.py`) +
`python -m app.scripts.test_promotions_rules` como verificación directa de las funciones puras.
Frontend: Karma/Jasmine (`*.spec.ts`).

**Target Platform**: API en Linux; frontend web responsive — el menú QR se consume en móvil, que
es la restricción que motiva el resumen a tres nombres (FR-002).

**Project Type**: Web application, dos repositorios independientes ya en producción.

**Performance Goals**: sin regresión de latencia en `GET /menu` y `GET /menu/promotions`. La
resolución de nombres añade **una consulta de coste constante** por llamada (`SELECT` sobre
`ProductVariant JOIN Product WHERE id IN (…)` con los ids de los conjuntos vigentes) — nunca un
N+1 por producto ni por regla.

**Constraints**: el importe cobrado no cambia en ningún escenario (SC-006); el criterio de
vigencia "en este instante" no cambia (A-57 intacto); todo el texto visible en español de
Colombia (Principio XIII).

**Scale/Scope**: 2 repositorios, ~17 ficheros tocados (5 de ellos nuevos, todos de test o
utilidades), 0 migraciones. Backend: `promotions/service.py`,
`menu/schemas.py`, `menu/router.py`. Frontend: `product.interface.ts`, `menu.service.ts`,
`diner.service.ts`, `product-select.component.ts`, `public-menu.component.ts`, un util nuevo
`promotion-condition.util.ts` y `promotions-page.component.ts`.

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

Evaluado contra la [Constitución v3.0.0](../../.specify/memory/constitution.md).

| Principio | Estado | Evidencia |
|---|---|---|
| I. Las nuevas funcionalidades nacen de un spec | ✅ | [spec.md](./spec.md) aprobada, con Clarifications cerradas el 2026-09-01. |
| **II. El comportamiento existente sigue protegido** | ⛔ **NO SATISFECHO** | Los tres cambios de comportamiento visible (A-66, A-67, A-68) **no están registrados**: el último asiento de [`registro-de-anomalias.md`](../000-reconocimiento/registro-de-anomalias.md) es **A-65**. Ver "Gate bloqueante" abajo. |
| III. Los characterization tests protegen el comportamiento heredado | ✅ con condición | **Ninguno** de los ficheros afectados contiene el prefijo bajo veto `"CONGELA comportamiento actual:"` (verificado por `grep` sobre los cuatro ficheros el 2026-09-01). Se pueden actualizar citando esta spec. **La spec lista 4 tests; el reconocimiento encontró un 5.º** — ver research.md D-8. |
| IV. Los nuevos specs pueden introducir nuevo comportamiento | ✅ | El criterio de éxito es la conformidad con esta spec, no la equivalencia con producción. |
| V. Nuevas funcionalidades antes que refactorizaciones oportunistas | ✅ | Cada cambio se ata a un FR. La única extracción (`promotion-condition.util.ts`) existe porque FR-018 exige el mismo algoritmo en un segundo punto del frontend, no como limpieza. |
| VI. Evolución incremental | ✅ | Cuatro historias entregables por separado, en orden P1 → P1 → P2 → P3; sin migraciones ni cambios de arquitectura mezclados. |
| VII. Compatibilidad con datos históricos | ✅ | Ninguna venta, factura ni pedido emitido se toca (FR-020). Nada de lo que se calcula aquí se persiste. |
| VIII. Evolución del modelo de datos | ✅ N/A | Sin entidades, campos, migración ni rollback de datos: FR-019 lo prohíbe y el diseño lo cumple. Todos los campos nuevos son de **respuesta de API**, no de esquema. |
| IX. Dependencias nuevas justificadas | ✅ N/A | Ninguna dependencia nueva. `unicodedata` es biblioteca estándar; en TypeScript la normalización usa `String.prototype.normalize`, nativa. |
| X. Verificación obligatoria | ✅ | [quickstart.md](./quickstart.md) define los escenarios ejecutables por criterio de aceptación, incluido el de no-regresión de importes (SC-006). |
| XI. Decisiones de negocio frente a técnicas | ✅ | Las tres decisiones de negocio están aisladas en A-66/A-67/A-68; las técnicas (dónde vive el cálculo, qué forma tiene el DTO) viven en research.md y en los contratos. |
| XII. Trazabilidad | ✅ | Necesidad (3 defectos reportados) → spec 066 → A-66/A-67/A-68 → tareas → tests → quickstart. |
| XIII. Todo en español de Colombia | ✅ | Artefactos, textos de UI y nombres de tests nuevos en español de Colombia. |

### Gate bloqueante (Principio II)

**A-66, A-67 y A-68 no existen en el registro de anomalías.** La propia spec lo declara paso
previo externo: *"Sin ese registro, el cambio de comportamiento no está autorizado"*
([spec.md](./spec.md), "Cómo se registran"). Consecuencias operativas:

- **Este plan y sus artefactos de diseño se entregan igual**: el diseño no cambia según quién
  firme la decisión, y tenerlo listo es lo que permite que el registro se escriba con el impacto
  ya conocido.
- **La implementación no puede arrancar** hasta que las tres entradas estén escritas a mano en
  [`registro-de-anomalias.md`](../000-reconocimiento/registro-de-anomalias.md), en el formato de
  A-62 a A-65 (qué cambia, por qué, quién decide, cuándo, funcionalidades afectadas).
- `/speckit-tasks` debe emitir esa verificación como **T001, tarea de arranque bloqueante**, tal
  como pide la spec ("las tareas solo verifican que existen").

Los tres asientos pendientes, con el alcance ya acotado por este plan:

| Entrada | Qué autoriza | Superficies afectadas |
|---|---|---|
| **A-66** | El texto de condición pasa de conteo (`"de estas 8 variantes"`) a nombres (`"Pequeño 8oz"`). | Cartel del menú QR, terminal, listado y formulario de administración. 5 tests a actualizar. |
| **A-67** | La insignia de la tarjeta del menú QR pasa de `🏷️ -10%` / `🏷️ -$2.000` (por tipo) a `🎉 Promo` (genérica). | Solo menú QR. La insignia de la terminal **no** cambia (FR-016). |
| **A-68** | `package_price` con `min_qty == 1` pasa a mostrarse como precio unitario vigente. | Menú QR: modal, tarjeta y total del botón de agregar. Corrige una discrepancia **mostrado ≠ cobrado**. |

### Re-evaluación post-diseño (Phase 1)

Repasada tras generar data-model, contratos y quickstart. **Sin violaciones nuevas.** Dos puntos
que el diseño confirmó en vez de introducir:

- **VIII (modelo de datos)**: el diseño cierra en cero cambios de esquema. `MenuVariantPromotion`
  es un DTO de respuesta; `Descriptor del conjunto` se calcula y se descarta.
- **V (sin refactors oportunistas)**: el `names` obligatorio de `variant_set_condition_text` es un
  cambio de firma, no un refactor: lo exige FR-001 y toca exactamente los 2 call sites que
  producen el texto.

El único gate en rojo sigue siendo el **II**, y es externo a esta feature.

## Project Structure

### Documentation (this feature)

```text
specs/066-promociones-legibles-menu/
├── plan.md                              # Este fichero
├── research.md                          # Phase 0 — decisiones técnicas y hallazgos del código
├── data-model.md                        # Phase 1 — entidades derivadas (ninguna persistida)
├── quickstart.md                        # Phase 1 — guía de validación ejecutable
├── contracts/
│   ├── texto-condicion.md               # FR-001 a FR-006 — el algoritmo, normativo
│   ├── menu-info-promocion.md           # FR-007 a FR-012 — el DTO y su cálculo
│   └── superficies.md                   # FR-013 a FR-018 — qué pinta cada superficie
├── checklists/                          # Ya existente
└── tasks.md                             # Phase 2 (/speckit-tasks — NO lo crea /speckit-plan)
```

### Source Code (repositorios de código)

```text
../pos-backend/
├── app/api/v1/promotions/service.py     # variant_set_condition_text(rule, names) + descriptor
│                                        #   + menu_variant_promotion() + variant_display_names()
├── app/api/v1/menu/schemas.py           # MenuVariantPromotion (nuevo) + campo en MenuVariantResponse
├── app/api/v1/menu/router.py            # _build_menu / _build_menu_promotions pasan los nombres
├── app/characterization_tests/
│   ├── test_menu_router.py              # texto nuevo + promotion por variante + FR-010
│   ├── test_promotions_router.py        # condition_text nuevo (5.º test, no listado en la spec)
│   ├── test_promotions_rules_admin.py   # condition_text nuevo en el alta
│   └── test_promociones_legibles.py     # NUEVO — aceptación de esta spec
└── app/scripts/test_promotions_rules.py # _regla_texto pasa a suministrar nombres

../pos-heladeria/
├── src/app/modules/products/interfaces/product.interface.ts    # MenuVariantPromotion en MenuVariant
├── src/app/core/services/menu.service.ts                       # mapea `promotion` (NO discounted_price)
├── src/app/modules/tables/services/diner.service.ts            # mapea `promotion` + los ya existentes
├── src/app/modules/tables/components/product-select.component.ts  # COMPARTIDO: comensal + cajero
├── src/app/modules/tables/pages/public-menu.component.ts       # insignia genérica 🎉 Promo
├── src/app/modules/promotions/services/
│   ├── promotion-condition.util.ts       # NUEVO — descriptor + textos (FR-002..FR-004) en TS
│   └── promotion-condition.util.spec.ts  # NUEVO
└── src/app/modules/promotions/pages/promotions-page.component.ts  # ruleConditionPreview con nombres
```

**Structure Decision**: se conserva la separación de los dos repositorios en producción, sin
crear módulos ni capas nuevas. El único fichero nuevo del frontend
(`promotion-condition.util.ts`) existe porque FR-018 obliga a replicar el algoritmo del backend en
la vista previa del formulario — el resto de superficies consume el texto que llega por la API,
así que no hay una segunda implementación escondida.

## Fases de entrega (Principio VI)

Cada fase es verificable por separado y corresponde a una historia de la spec.

| Fase | Historia | Alcance | Se puede desplegar sola |
|---|---|---|---|
| **0** | — | Verificar A-66/A-67/A-68 en el registro. **Bloqueante.** | — |
| **1** | US1 (P1) | Descriptor por nombres + `variant_set_condition_text(rule, names)` + los 2 call sites + 5 tests actualizados. | Sí: el cartel del menú, el listado de administración y la terminal (que ya lee `condition_text`) mejoran de golpe. |
| **2** | US2 (P1) | `MenuVariantPromotion` en la respuesta del menú + `discounted_price` para `package_price` `min_qty == 1` (FR-010) + pintado en `ProductSelectComponent`. | Sí. Es la fase que corrige el defecto de importe. |
| **3** | US3 (P2) | Insignia genérica `🎉 Promo` en la tarjeta del menú QR, derivada de `variants[].promotion`. | Sí. Depende de la fase 2 (necesita el campo). |
| **4** | US4 (P3) | `ruleConditionPreview` sobre nombres seleccionados + `MenuService` mapea `promotion` para la terminal. | Sí. |

## Complexity Tracking

> Sin violaciones del Constitution Check que justificar. El único gate en rojo (Principio II,
> A-66/A-67/A-68 sin registrar) no es una complejidad de diseño que haya que aceptar: es un paso
> previo externo pendiente, y se resuelve escribiendo las tres entradas.
