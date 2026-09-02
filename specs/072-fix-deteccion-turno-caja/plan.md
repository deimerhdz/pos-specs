# Implementation Plan: Corrección — la Terminal de mesas no detecta un turno de caja que sí está abierto

**Branch**: `072-fix-deteccion-turno-caja` | **Date**: 2026-09-02 | **Spec**: [spec.md](./spec.md)

**Input**: Feature specification from `/specs/072-fix-deteccion-turno-caja/spec.md`

## Summary

Corrección 100% frontend (`pos-heladeria`), sin ningún cambio en `pos-backend`. El reconocimiento
de código (spec.md + research.md) ya aisló las dos causas exactas y su fix:

1. **Disparador reactivo (US1, FR-005 a FR-007)** — `PosTerminalStore.reloadOrders()`
   ([pos-terminal.store.ts:1074-1081](../../../pos-heladeria/src/app/modules/tables/services/pos-terminal.store.ts#L1074-L1081)),
   el único funnel de la carga inicial, el sondeo y los eventos en tiempo real, gana una llamada
   condicional a `ensureCheckoutDataLoaded()` (ya existente) cuando hay algo real para cobrar
   (`hasChargeableOrderNow()`, nuevo método privado que reutiliza el umbral que ya usan
   `selectTable()`/`selectStandaloneOrder()`, sin duplicarlo). Deliberadamente **no** es un
   `effect()` reactivo sobre `orders()`/`pendingOfSelectedTable()`: ese patrón ya rompió "decenas
   de specs" la primera vez que se intentó en este mismo archivo
   (comentario en [líneas 267-280](../../../pos-heladeria/src/app/modules/tables/services/pos-terminal.store.ts#L267-L280)),
   y los 35 usos actuales de `store.orders.set([...])` en los tests de este store confirman que el
   riesgo sigue vigente — enganchar dentro del método, no en la reactividad del signal, los deja
   intactos ([contracts/disparador-reactivo-cobro.md](./contracts/disparador-reactivo-cobro.md)).
2. **Descubrimiento sin depender de `localStorage` (US2/US3, FR-001 a FR-004)** —
   `CashService.restoreShift()`
   ([cash.service.ts:50-63](../../../pos-heladeria/src/app/modules/cash-register/services/cash.service.ts#L50-L63))
   se reemplaza por `discoverOpenShift()`: camino rápido con `localStorage` si ya resuelve algo, y
   si no, lista las cajas del tenant y consulta el turno actual de cada una (mismo patrón que ya
   usa `CashSessionStore.loadOverview()`) — exactamente un turno abierto se adopta automáticamente;
   cero seguirán bloqueado; dos o más siguen exigiendo "Operar" explícito, sin adivinar
   ([contracts/descubrimiento-turno-abierto.md](./contracts/descubrimiento-turno-abierto.md)).

Ningún endpoint de `pos-backend` cambia — `GET /cash/registers` y
`GET /cash/shifts/current?cash_register_id=` ya son suficientes (confirmado en research.md D3 que
`current_shift` no tiene ningún filtro por usuario/sesión que pudiera explicar el falso negativo
de otra forma).

## Technical Context

**Language/Version**: TypeScript 5.x / Angular 20 standalone, signals (`pos-heladeria`) — sin
cambio de versión. `pos-backend` no se toca en esta spec.

**Primary Dependencies**: sin cambio, sin dependencia nueva (Principio IX: nada que justificar).

**Storage**: PostgreSQL 16, schema-per-tenant. **Sin cambio de esquema y sin migración**
([data-model.md](./data-model.md)) — se reutilizan las tablas `cash_registers`/`cash_shifts` tal
cual.

**Testing**: Karma/Jasmine → ahora Vitest (`*.spec.ts`, `npm test`), sin cambio de framework.
`python -m unittest discover -s app/characterization_tests -p 'test_*.py' -v` en backend, como red
de seguridad (no se espera ningún impacto).

**Target Platform**: frontend web, Terminal de mesas (uso en dispositivos de caja/mesero).

**Project Type**: Web application, dos repositorios independientes ya en producción — esta spec
solo toca `pos-heladeria`.

**Performance Goals**: el chequeo de turno se reintenta en cada `reloadOrders()` mientras algo
siga pendiente de cobrar y el turno no se haya resuelto (research.md D4) — costo acotado (1-3
peticiones en paralelo, típico de un tenant con una a tres cajas) y transitorio (deja de repetirse
en cuanto se resuelve). Margen aceptado de hasta ~2 segundos entre que el panel de cobro aparece y
el control refleja el estado correcto (Clarifications, sesión 2026-09-02).

**Constraints**: la spec 059 FR-001 (no pedir datos de cobro con solo una mesa libre sin pedido)
se mantiene intacta — el nuevo disparador reutiliza el mismo umbral "hay algo real que cobrar" que
ya usan los disparadores imperativos existentes, no lo relaja. Con dos o más turnos abiertos a la
vez, el sistema sigue sin adivinar (FR-004, User Story 3) — riesgo financiero de atribuir un cobro
a la caja equivocada.

**Scale/Scope**: 1 repositorio, 0 endpoints nuevos, 0 migraciones. Archivos de producción: 2
(`cash.service.ts`, `pos-terminal.store.ts`) + 1 call site en `cash-session.store.ts`. Tests:
extender `cash.service.spec.ts` (sin tests previos de `restoreShift`, ninguno que romper) y el
describe "carga diferida... (spec 059, Historia 1)" de `pos-terminal.store.spec.ts`; actualizar
dos *stubs* de mock (`pos-terminal.store.spec.ts:52`, `pos-tables-panel.component.spec.ts:65`) que
hoy exponen `restoreShift` en vez de `discoverOpenShift`.

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

Evaluado contra la [Constitución v3.0.0](../../.specify/memory/constitution.md).

| Principio | Estado | Evidencia |
|---|---|---|
| I. Las nuevas funcionalidades nacen de un spec | ✅ | [spec.md](./spec.md) aprobada, Clarifications cerradas el 2026-09-02. |
| II. El comportamiento existente sigue protegido | ✅ N/A | Es una corrección de un defecto, no un cambio de regla de negocio deliberada — spec.md ya lo declara explícitamente en "Autorización de negocio"; no aplica entrada nueva en `registro-de-anomalias.md`. |
| III. Los characterization tests protegen el comportamiento heredado | ✅ | Ningún test relacionado (`cash.service.spec.ts`, `cash-session.store.spec.ts`, `pos-terminal.store.spec.ts`, `test_cash_timezone.py`) lleva el prefijo `"CONGELA comportamiento actual:"` (research.md D5). Los 35 usos de `orders.set(...)` fuera de `reloadOrders()` quedan intactos por diseño (research.md D1). |
| IV. Los nuevos specs pueden introducir nuevo comportamiento | ✅ N/A | No aplica — es una corrección, el criterio de éxito es igualar el comportamiento pretendido (spec 028/059), no introducir uno nuevo. |
| V. Nuevas funcionalidades antes que refactorizaciones oportunistas | ✅ | Cada cambio se ata a un FR. `discoverOpenShift()` reutiliza literalmente el patrón ya existente de `loadOverview()`, no inventa uno nuevo; `hasChargeableOrderNow()` extrae una condición ya duplicada en dos sitios, no diseña una nueva. |
| VI. Evolución incremental | ✅ | Un solo repositorio tocado, 0 migraciones, 0 endpoints nuevos — no mezcla clases de cambio. |
| VII. Compatibilidad con datos históricos | ✅ N/A | Ninguna venta, factura ni pedido se toca. El fix es puramente sobre cómo se detecta un turno ya abierto, antes de cobrar — no reescribe nada ya cobrado. |
| VIII. Evolución del modelo de datos | ✅ N/A | Cero cambios de esquema, cero migraciones ([data-model.md](./data-model.md)). |
| IX. Dependencias nuevas justificadas | ✅ N/A | Ninguna dependencia nueva. |
| X. Verificación obligatoria | ✅ | [quickstart.md](./quickstart.md) define los escenarios ejecutables por historia, incluida la verificación explícita de que los 35 tests existentes no se rompen. |
| XI. Decisiones de negocio frente a técnicas | ✅ | No hay decisión de negocio nueva que tomar — spec.md ya lo resuelve; las decisiones técnicas (dónde enganchar el disparador, cómo descubrir el turno) viven en research.md D1-D4. |
| XII. Trazabilidad | ✅ | Necesidad (bug reportado con capturas) → spec 072 → Clarifications → este plan → contratos → tasks.md → tests → quickstart. |
| XIII. Todo en español de Colombia | ✅ | Sin textos de UI nuevos que agregar (el mensaje de error ya existente no cambia su redacción, solo deja de mostrarse cuando no debería). |

### Re-evaluación post-diseño (Phase 1)

Repasada tras generar data-model, los dos contratos y quickstart. **Sin violaciones nuevas.**

- **III confirmado con más precisión**: research.md D1 no solo declaró "no hay tests protegidos"
  en general — identificó el riesgo concreto (35 usos de `orders.set(...)`) y el diseño (enganchar
  en `reloadOrders()`, no en un `effect()`) que lo neutraliza por construcción, no por suerte.
  Sigue quedando como tarea de verificación correr la suite completa y confirmar 0 regresiones.
- **V confirmado**: el diseño de Phase 1 no agregó ningún patrón nuevo — ambos contratos citan
  explícitamente el código ya existente que reutilizan (`loadOverview()`, el umbral de
  `selectTable()`).

Ningún gate queda en rojo.

## Project Structure

### Documentation (this feature)

```text
specs/072-fix-deteccion-turno-caja/
├── plan.md              # Este archivo
├── research.md          # Decisiones D1-D5
├── data-model.md         # Cero cambios de esquema; qué signal cambia de comportamiento
├── contracts/
│   ├── descubrimiento-turno-abierto.md   # FR-001 a FR-004
│   └── disparador-reactivo-cobro.md      # FR-005 a FR-007
├── quickstart.md        # Validación ejecutable
└── tasks.md              # Salida de /speckit-tasks (no de este comando)
```

### Source Code (repository root)

```text
../pos-heladeria/
└── src/app/modules/
    ├── cash-register/services/
    │   ├── cash.service.ts             # restoreShift() → discoverOpenShift()
    │   ├── cash.service.spec.ts        # tests nuevos: camino rápido, descubrimiento, 0/1/N turnos
    │   └── cash-session.store.ts       # call site de discoverOpenShift() (rama admin, línea ~251)
    └── tables/
        ├── services/
        │   ├── pos-terminal.store.ts       # reloadOrders() + hasChargeableOrderNow() + call site renombrado
        │   └── pos-terminal.store.spec.ts  # tests nuevos en el describe "carga diferida (spec 059, Historia 1)"
        └── components/
            └── pos-tables-panel.component.spec.ts   # stub de mock: restoreShift → discoverOpenShift
```

**Structure Decision**: cero archivos nuevos de producción — se extienden dos servicios/stores ya
existentes. `pos-backend` no aparece en este árbol porque no se toca.

## Complexity Tracking

> Sin violaciones del Constitution Check que justificar — tabla omitida.
