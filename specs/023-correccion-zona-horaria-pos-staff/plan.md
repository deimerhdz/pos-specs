# Implementation Plan: Corrección de zona horaria en el POS de staff (previsualización de promociones) (A-09)

**Branch**: `023-correccion-zona-horaria-pos-staff` | **Date**: 2026-08-18 | **Spec**: [spec.md](./spec.md)

**Input**: Feature specification from `/specs/023-correccion-zona-horaria-pos-staff/spec.md`

## Summary

`pos-terminal.store.ts` construye `now = new Date()` en cuatro sitios (líneas 248, 262, 386, 1190)
y se lo pasa a las funciones puras de `promotion-pricing.util.ts`
(`isPromoActiveNow`/`bestProductDiscount`/`discountedUnitPrice`), que **no cambian** — reciben `now`
como parámetro y ya son correctas dado un valor válido. El defecto es enteramente de dónde sale ese
`now`: el reloj crudo del dispositivo, sin ninguna conversión a `TENANT_TIMEZONE`, porque el cliente
nunca ha recibido esa hora en ningún endpoint que consuma.

Esta spec corrige eso agregando la hora del servidor a la respuesta del único endpoint que el POS
de staff ya sondea para promociones (`GET /promotions?status=active`, vía
`PromotionService.activeQuery`): el backend expone un header `X-Server-Time` (UTC, ISO 8601) en
`list_promotions` (`app/api/v1/promotions/router.py:37`); el frontend, en el mismo `queryFn` que ya
llama a ese endpoint, calcula el desfase entre esa hora y `Date.now()` local
(`PromotionService.serverTimeOffsetMs`) y expone `PromotionService.now()` como reemplazo de
`new Date()`. Los cuatro sitios de `pos-terminal.store.ts` pasan a llamar
`this.promotionService.now()`. Mientras no haya llegado la primera respuesta (offset desconocido),
el POS degrada explícito: no evalúa vigencia con un reloj no verificado, sencillamente no muestra
insignias ni descuentos de previsualización hasta el primer sync (FR-003/FR-004).

No se toca `promotion-pricing.util.ts` (funciones puras), ni el motor de promociones del backend
(`active_discount_promotions`/`local_now`/`best_line_discount`, spec 012, A-07 protegida), ni el
desempate del frontend (A-10). El header es aditivo y exclusivo de `GET /promotions`: no se toca
`Page[T]` (`app/core/pagination.py`, genérico compartido por decenas de endpoints).

## Technical Context

**Language/Version**: backend Python 3.14 (venv de `pos-backend`, `env/pyvenv.cfg`, mismo que spec
022); frontend TypeScript 5.9.2 / Angular 21.1 (`pos-heladeria/package.json`)

**Primary Dependencies**: backend — FastAPI + `starlette.responses.Response` (ya en uso, ningún
import nuevo salvo `datetime`/`timezone` de stdlib en `promotions/router.py`, que hoy no los
importa). Frontend — `@angular/common/http` (ya en uso por `PromotionService`), Angular `signal`
(ya en uso); ningún paquete nuevo (Constitución, Principio IV: no aplica justificación porque no se
añade ninguna dependencia)

**Storage**: N/A — el cambio es de qué dato viaja en una respuesta HTTP ya existente y de qué reloj
se usa para una evaluación de solo lectura en el cliente; ningún esquema de base de datos ni de
`localStorage`/`sessionStorage` cambia

**Testing**: backend — `unittest` vía `python -m unittest`, invocando la función del endpoint
directamente como función Python (mismo patrón que
`app/characterization_tests/test_table_sessions_router.py` — `Depends(...)` nunca se resuelve
llamando la función a mano, así que basta pasar `response=Response()`, `db`, `_=` doble de usuario,
y leer `response.headers` después). No existe hoy ningún `test_promotions_router.py`; esta spec crea
el primero, acotado a la cabecera `X-Server-Time`. Frontend — Vitest (`pos-heladeria/package.json`,
`"vitest": "^4.0.8"`) vía `TestBed` + `provideHttpClientTesting()`/`HttpTestingController` +
`provideTanStackQuery()`, mismo harness que `product.service.spec.ts`. `pos-terminal.store.spec.ts`
hoy **no** instancia `PosTerminalStore` (solo prueba funciones puras exportadas,
`deriveTableStatus`/`newPendingIds`) — no existe ningún test previo que congele el defecto de A-09
en los cuatro sitios; ver research.md Decisión 3 para el límite de prueba elegido

**Target Platform**: backend Linux server (`pos-backend`, API FastAPI en producción); frontend
navegador de terminal POS (`pos-heladeria`, Angular SPA, `api.skeilopos.com` en origen distinto al
de cada subdominio de tenant — CORS real, no same-origin)

**Project Type**: corrección puntual cross-repo (backend + frontend), sin extracción de módulo ni
paquete nuevo — un header aditivo en un endpoint existente, y un punto de invocación corregido en
un store frontend existente

**Performance Goals**: sin objetivo nuevo — un header adicional (~30 bytes) en una respuesta que ya
viaja, y un cálculo de resta de dos `number` en cliente; sin llamada de red adicional (RF2)

**Constraints**: no se toca `promotion-pricing.util.ts` (`isPromoActiveNow`/`inTimeWindow`/
`bestProductDiscount`/`matchingTarget`/`packTerms`/`lineDiscount`/`findOverlaps`) — Principio III;
no se toca el motor de promociones del backend (`app/api/v1/promotions/service.py`, spec 012, A-07
protegida); no se toca `app/core/pagination.py` (`Page[T]`, genérico compartido) — el header va solo
en la respuesta de `GET /promotions`, no en el modelo; no se recalcula ninguna promoción, pedido ni
factura ya generados (FR-006); el criterio de desempate A-10 (`bestProductDiscount`) no cambia
(FR-005)

**Scale/Scope**: backend — 1 import + ~2 líneas en `promotions/router.py` (`list_promotions`) + 1
línea en `main.py` (`expose_headers`) + 1 fichero de test nuevo (`test_promotions_router.py`).
Frontend — ~10 líneas nuevas en `promotion.service.ts` (signal de offset, método `now()`, computed
`ready`, lectura del header dentro del `queryFn` de `activeQuery`) + 4 sustituciones de `new Date()`
por `this.promotionService.now()` en `pos-terminal.store.ts` (líneas 248, 262, 386, 1190) + guardas
de "aún sin sync" en esos mismos 4 sitios + 1 fichero de test nuevo
(`promotion.service.now.spec.ts` o ampliación de un spec existente, ver research.md Decisión 3)

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| Principio | Evaluación | Estado |
|---|---|---|
| **I. El Comportamiento Sigue Siendo Sagrado (por Defecto)** | El cambio de comportamiento reabre una decisión previamente cerrada de A-09 ("mitigado operativamente, documentar sin especificar"). La reapertura está autorizada por escrito y fechada en `registro-de-anomalias.md` (nota "Actualización — reapertura de la decisión, 2026-08-18") y citada en `spec.md` §Autorización de negocio y `FR-007`. Ningún otro comportamiento cambia por criterio técnico. | PASS |
| **II. Los Characterization Tests son el Árbitro** | No existía ningún test que congelara el defecto de A-09 (verificado: ni `pos-terminal.store.spec.ts` ni `promotion-pricing.util.spec.ts` lo cubrían). Esta spec crea tests nuevos en ambos repos citando A-09 (FR-008), sin modificar ningún test `CONGELA` existente de otra anomalía. `app/scripts/test_promotions_rules.py` (único script en CI) no importa `promotions/router.py` ni se ve afectado. | PASS |
| **III. Estrangulamiento antes que Reescritura** | Un endpoint (`list_promotions`) y un store (`PosTerminalStore`) ya existentes, ambos con un cambio puntual y localizado — no hay extracción ni reescritura de módulo. `promotion-pricing.util.ts`, `promotions/service.py` (backend) y `Page[T]` quedan explícitamente sin tocar (Out of Scope de `spec.md`). | PASS |
| **IV. Dependencias Nuevas Permitidas con Justificación** | No se añade ninguna dependencia de terceros en ningún repo. Único import nuevo: `datetime`/`timezone` (stdlib) en `promotions/router.py`, que hoy no los usa — no requiere justificación de la Constitución (no es una dependencia externa). | PASS (no aplica) |
| **V. Ningún Cambio Retroactivo** | FR-006 lo exige explícitamente; `list_promotions` y las cuatro `computed`/método de `pos-terminal.store.ts` son de solo lectura, recalculadas en cada render/petición — el cambio solo afecta lo que se muestra o se sirve **después** del despliegue. Ninguna promoción, pedido ni factura ya generados se toca. | PASS |
| **VI. Todo en Español de Colombia** | Esta spec, este plan y los artefactos que genera (research.md, data-model.md, contracts/, quickstart.md) se escriben en español de Colombia, igual que las specs 007/012/020/021/022. | PASS |

Sin violaciones. La tabla de Complexity Tracking al final de este documento queda vacía.

## Project Structure

### Documentation (this feature)

```text
specs/023-correccion-zona-horaria-pos-staff/
├── plan.md              # Este fichero (/speckit-plan)
├── research.md          # Fase 0 (/speckit-plan) — mecanismo del header, dónde vive el offset,
│                           límite de prueba elegido, comportamiento antes del primer sync
├── data-model.md         # Fase 1 (/speckit-plan)
├── quickstart.md          # Fase 1 (/speckit-plan)
├── contracts/
│   └── promotions-list-endpoint.md   # Fase 1 (/speckit-plan) — GET /promotions, header nuevo
└── tasks.md               # Fase 2 (/speckit-tasks — NO se crea en este comando)
```

### Source Code (dos repos sibling de `pos-specs`)

Esta spec vive en `pos-specs`, pero el código que describe está en `../pos-backend` y
`../pos-heladeria` (siblings de este repo, según la Constitución §Alcance). Rutas relativas a la
raíz de cada uno.

```text
# ../pos-backend
app/
├── main.py                          # CAMBIA — agrega "X-Server-Time" a expose_headers de
│                                        CORSMiddleware (línea ~95, junto a ETag/Retry-After)
│
├── api/v1/promotions/
│   ├── router.py                    # CAMBIA — list_promotions:37 recibe `response: Response`,
│   │                                    escribe response.headers["X-Server-Time"] con
│   │                                    datetime.now(timezone.utc).isoformat() (FR-001/FR-002
│   │                                    del lado servidor); resto de la función SIN CAMBIOS
│   └── service.py                   # SIN CAMBIOS — active_discount_promotions/local_now/
│                                        best_line_discount, contrato ya fijado por spec 012
│                                        (A-07 protegida)
│
├── core/pagination.py                # SIN CAMBIOS — Page[T] no se toca (ver Constraints)
│
└── characterization_tests/
    └── test_promotions_router.py     # NUEVO — primer characterization test de
                                          promotions/router.py, acotado a la cabecera
                                          X-Server-Time y A-09

# ../pos-heladeria
src/app/modules/
├── promotions/services/
│   ├── promotion.service.ts          # CAMBIA — activeQuery.queryFn pasa a observe:'response',
│   │                                     lee X-Server-Time, actualiza serverTimeOffsetMs (signal
│   │                                     nueva); agrega now(): Date y ready: Signal<boolean>
│   │                                     (FR-001/FR-002/FR-004)
│   ├── promotion.service.spec.ts     # NUEVO o ampliado — characterization de now()/ready/
│   │                                     offset, con HttpTestingController simulando el header
│   └── promotion-pricing.util.ts     # SIN CAMBIOS — isPromoActiveNow/inTimeWindow/
│                                         bestProductDiscount/discountedUnitPrice, funciones puras
│                                         ya correctas dado un `now` válido
│
└── tables/services/
    ├── pos-terminal.store.ts         # CAMBIA — 4 sitios (líneas 248, 262, 386, 1190):
    │                                     new Date() → this.promotionService.now(), con guarda
    │                                     de this.promotionService.ready() antes de evaluar
    │                                     vigencia (FR-003/FR-004); resto del store SIN CAMBIOS
    └── pos-terminal.store.spec.ts    # AMPLIADO — characterization de los 4 puntos corregidos,
                                          citando A-09 (research.md Decisión 3 fija el alcance
                                          exacto de qué se prueba con TestBed vs. qué queda
                                          delegado a promotion.service.spec.ts)
```

**Structure Decision**: dos repos, cuatro ficheros de producción cambiados (uno de config CORS, uno
de router backend, uno de servicio frontend, uno de store frontend), sin paquete ni módulo nuevo en
ninguno de los dos — no es una extracción. Los únicos artefactos nuevos son
`test_promotions_router.py` (backend, porque hoy no existe ninguna caracterización de
`promotions/router.py`) y el test nuevo/ampliado del lado frontend para `now()`/`ready` y los 4
puntos de invocación corregidos.

## Complexity Tracking

> **Fill ONLY if Constitution Check has violations that must be justified**

Sin violaciones — tabla vacía.
