# Implementation Plan: Corrección — "Liberar Mesa" bloqueada por un pedido ya cancelado

**Branch**: `050-correccion-liberar-mesa-pedido-cancelado` | **Date**: 2026-08-29 | **Spec**: [spec.md](./spec.md)

**Input**: Feature specification from `/specs/050-correccion-liberar-mesa-pedido-cancelado/spec.md`

## Summary

Dos correcciones independientes, una por repositorio:

1. **Backend** (`pos-backend`): en `release_paid_session`
   (`app/api/v1/table_sessions/service.py:337-370`), el query que arma la lista de pedidos para
   `_assert_closable` (líneas ~366-369) agrega `CustomerOrder.status != "cancelada"` a su `where`,
   igualándose al criterio que `_billable_orders`/`has_billable_orders` ya aplican — sin tocar
   `_assert_closable` en sí, ni la lógica de inventario de `cancel_order`
   (`app/api/v1/orders/checkout.py:519-566`), que deliberadamente no anula `estado_cocina` al
   cancelar un pedido.
2. **Frontend** (`pos-heladeria`): en `ToastService.push()`
   (`src/app/shared/feedback/toast.service.ts:36-42`), se agrega una revisión previa: si ya existe
   un toast con el mismo `kind` y `text`, no se agrega uno nuevo. Sin cambios de firma en
   `success`/`error`/`info`.

Ver `research.md` (decisiones D1-D2) y `contracts/api-contract.md` para el detalle exacto de cada
comportamiento antes/después.

## Technical Context

**Language/Version**: Backend: Python 3.14.4, FastAPI 0.136.3, SQLAlchemy (ORM), PostgreSQL 16
(schema-per-tenant). Frontend: TypeScript 5.9.2, Angular 21.1 (componentes standalone, signals)

**Primary Dependencies**: Sin dependencias nuevas en ninguno de los dos repositorios — el fix de
backend es un filtro adicional en un `select(...)` ya existente (SQLAlchemy ya en uso); el de
frontend es una condición sobre un array ya en memoria (`signal<Toast[]>`)

**Storage**: PostgreSQL (backend) — sin cambios de esquema, sin migración; solo cambia qué filas
califican en un query de lectura ya existente. Frontend: sin storage, `toasts` vive en memoria de
la pestaña

**Testing**: Backend: `pytest` sobre `app/characterization_tests/` (patrón `unittest.TestCase` +
fixtures de `table_sessions_fixtures.py`). Frontend: Vitest vía el builder
`@angular/build:unit-test`, specs colocados `*.service.spec.ts`

**Target Platform**: Backend: servidor Linux (API REST). Frontend: SPA Angular en navegador de
escritorio (Terminal de Mesas)

**Project Type**: Corrección sobre dos aplicaciones existentes en producción (`pos-backend` +
`pos-heladeria`) — sin cambios de arquitectura en ninguna

**Performance Goals**: Sin objetivos numéricos nuevos — un filtro adicional en un `WHERE` ya
indexado por `table_session_id`, y una búsqueda lineal sobre un array de toasts que en la práctica
nunca supera unas pocas entradas simultáneas

**Constraints**: Cero cambios en `_assert_closable` (sigue sirviendo a `close_session` sin cambios),
cero cambios en `cancel_order` ni en su lógica de ajuste de inventario (Principio V — no mezclar
esta corrección con esa lógica delicada, ver research.md D1), cero cambios en la firma pública de
`ToastService`; no se agregan dependencias nuevas (Principio IX)

**Scale/Scope**: Dos archivos de producción, uno por repositorio —
`pos-backend/app/api/v1/table_sessions/service.py` (un filtro de una línea en el `where` de
`release_paid_session`, más ajuste del docstring) y
`pos-heladeria/src/app/shared/feedback/toast.service.ts` (una condición de guarda en `push`). Un
test nuevo en `pos-backend/app/characterization_tests/test_table_sessions_service.py` (junto a los
`test_release_paid_session_*` ya existentes) y un archivo de test nuevo
`pos-heladeria/src/app/shared/feedback/toast.service.spec.ts` (no existe hoy ninguna cobertura de
`ToastService`). Ningún otro archivo ni endpoint se modifica.

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

- **Principio I (Nace de un spec)** ✅ — `specs/050-correccion-liberar-mesa-pedido-cancelado/spec.md`
  existe, sin `[NEEDS CLARIFICATION]` pendiente (las 2 decisiones de alcance se resolvieron con el
  dueño del producto antes de escribir la spec), antes de este plan.
- **Principio II (Comportamiento existente protegido)** ✅ — spec.md documenta explícitamente que es
  un bug preexistente de `pos-backend` (no introducido por ninguna spec previa de esta serie), con
  autorización directa del dueño/desarrollador el 2026-08-29; FR-002/FR-003 exigen explícitamente
  que el comportamiento ya correcto (pedidos `'pagada'` con cocina en curso, y cualquier otro motivo
  de bloqueo) no cambie. No aplica una nueva entrada en `registro-de-anomalias.md` más allá de citar
  esta spec como origen y autorización (mismo criterio que specs 045/048/049 para correcciones de
  navegación/validación, no de precio/inventario/facturación).
- **Principio III (Characterization tests)** ✅ — research.md confirma: 0 tests
  `"CONGELA comportamiento actual:"` dependen de que `release_paid_session` incluya pedidos
  cancelados, ni de que `ToastService` apile duplicados — ambos son huecos de cobertura, no
  comportamiento protegido; el test ya existente sobre pedidos `'pagada'`
  (`test_release_paid_session_409_con_cocina_en_curso_sobre_pedido_pagado`) sigue pasando sin
  cambios.
- **Principio IV (Nuevo comportamiento vía spec)** ✅ — el comportamiento nuevo (excluir pedidos
  cancelados del chequeo de cocina; deduplicar avisos) está definido en spec.md, FR-001 a FR-007.
- **Principio V (No refactors oportunistas)** ✅ — el fix de backend es un filtro agregado al
  `where` ya existente, sin tocar `_assert_closable` ni `cancel_order`/su lógica de inventario
  (research.md, D1, alternativa rechazada explícitamente); el de frontend es una guarda agregada a
  `push()`, sin tocar `success`/`error`/`info` ni ningún llamador.
- **Principio VI (Evolución incremental)** ✅ — dos correcciones independientes y pequeñas, cada una
  en un solo archivo de producción de un solo repositorio; sin migración de datos ni cambio de
  arquitectura.
- **Principio VII (Datos históricos)** N/A — no se toca facturación, ventas ni ningún dato ya
  emitido; el fix decide qué pedidos ya existentes califican en un query de lectura sobre el estado
  actual de la sesión, no reconstruye nada histórico.
- **Principio VIII (Evolución del modelo de datos)** N/A — data-model.md: sin entidades ni columnas
  nuevas; el cambio es sobre qué subconjunto de datos ya existentes usa una consulta ya existente.
- **Principio IX (Dependencias nuevas)** ✅ — no se agrega ninguna dependencia en ninguno de los dos
  repositorios.
- **Principio X (Verificación obligatoria)** — pendiente de ejecución en `/speckit-tasks` +
  `/speckit-implement`: agregar el test nuevo en `test_table_sessions_service.py` (mesa con pedido
  `'cancelada'` y cocina sin terminar → libera sin 409) confirmando que el existente sobre `'pagada'`
  sigue en verde; crear `toast.service.spec.ts` (cobertura nueva, no existe hoy) cubriendo
  deduplicación por texto+tipo, no-deduplicación entre textos/tipos distintos, y reaparición tras
  que el duplicado original ya desapareció.
- **Principio XI (Negocio vs. técnico)** ✅ — las 2 decisiones de negocio (corregir también el
  apilamiento de toasts; documentarlo como spec formal en vez de hotfix directo) ya quedaron
  registradas en spec.md, Clarifications, tomadas por el dueño del producto antes de este plan; las
  decisiones de este documento (D1-D2) son todas técnicas (cómo, no qué).
- **Principio XII (Trazabilidad)** ✅ — Necesidad (reporte de bug + captura de pantalla) → Spec 050
  → este Plan → Tasks/Implementación/Tests (siguientes comandos).
- **Principio XIII (Español de Colombia)** ✅ — todo este documento y los artefactos generados se
  redactan en español de Colombia.

**Resultado**: Gate PASA sin violaciones. No se requiere la tabla de Complexity Tracking.

**Re-chequeo post-diseño (tras Fase 1)**: `data-model.md` y `contracts/api-contract.md` confirman
que el diseño no agrega dependencias nuevas (Principio IX), no modifica el modelo de datos
(Principio VIII, N/A), no toca ningún characterization test existente (Principio III), y no altera
ningún dato histórico ni regla de facturación/inventario (Principio VII) — únicamente ajusta qué
pedidos ya existentes califican en dos validaciones de lectura ya existentes. Gate sigue PASANDO
sin violaciones.

## Project Structure

### Documentation (this feature)

```text
specs/050-correccion-liberar-mesa-pedido-cancelado/
├── plan.md              # Este archivo (/speckit-plan)
├── research.md          # Fase 0 (/speckit-plan) — decisiones D1-D2
├── data-model.md         # Fase 1 (/speckit-plan) — tabla de estados afectados
├── quickstart.md         # Fase 1 (/speckit-plan) — validación manual, 3 escenarios
├── contracts/            # Fase 1 (/speckit-plan) — contrato de comportamiento del endpoint + ToastService
│   └── api-contract.md
└── tasks.md              # Fase 2 (/speckit-tasks) — aún no generado
```

### Source Code (dos repositorios hermanos)

El código vive en dos repositorios hermanos de este `pos-specs`: `../pos-backend` (FastAPI) y
`../pos-heladeria` (Angular). No hay ningún componente ni servicio nuevo — ambos fixes modifican
funciones ya existentes.

```text
pos-backend/app/api/v1/table_sessions/
├── service.py                          # HOY: `release_paid_session` (~337-370) arma `orders` con
│                                         # un query sin filtro de `status` (~366-369) antes de
│                                         # `_assert_closable` → SE AGREGA `CustomerOrder.status !=
│                                         # "cancelada"` al `where`; `_assert_closable` (~217-241) y
│                                         # `_billable_orders`/`has_billable_orders` (~65-157) no
│                                         # cambian
└── router.py                           # Sin cambios — mismo endpoint, mismo schema

pos-backend/app/characterization_tests/
└── test_table_sessions_service.py      # Se agrega un test junto a los `test_release_paid_session_*`
                                          # ya existentes (~línea 647)

pos-heladeria/src/app/shared/feedback/
├── toast.service.ts                    # HOY: `push()` (~36-42) agrega siempre → SE AGREGA una
│                                         # guarda: si ya existe `{kind, text}` igual en `toasts()`,
│                                         # no agrega; `success`/`error`/`info` sin cambios de firma
└── toast.service.spec.ts               # Nuevo — no existe cobertura hoy de `ToastService`
```

**Structure Decision**: Se modifican in-place dos archivos de producción ya existentes, uno por
repositorio (`service.py` en `pos-backend`, `toast.service.ts` en `pos-heladeria`), sin crear
ningún módulo, servicio ni componente nuevo. Los dos fixes son independientes entre sí (repos y
capas distintas) y pueden implementarse y verificarse en cualquier orden.

## Complexity Tracking

*Sin violaciones que justificar — el Constitution Check pasa limpio (ver arriba).*
