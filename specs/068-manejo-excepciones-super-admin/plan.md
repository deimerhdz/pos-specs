# Implementation Plan: Manejo de excepciones y respuestas de error consistentes en el módulo super-admin

**Branch**: `068-manejo-excepciones-super-admin` | **Date**: 2026-09-02 | **Spec**: [spec.md](./spec.md)

**Input**: Feature specification from `/specs/068-manejo-excepciones-super-admin/spec.md`

**Repositorio de implementación**: `../pos-backend` (este repositorio, `pos-specs`, solo contiene la
especificación; el código vive en el repositorio hermano `pos-backend`, FastAPI + PostgreSQL
16 schema-per-tenant, según la constitución).

## Summary

Estandarizar cómo el módulo `app/api/v1/super_admin/` (router principal + sub-routers de
catálogo de planes y de métodos de pago) comunica sus errores: una capa de dominio nueva y
mínima de excepciones sin dependencia de FastAPI, un middleware de traducción a HTTP con
alcance exclusivo al prefijo `/api/v1/super-admin` (así ningún otro módulo cambia de
comportamiento), un envelope de error consistente y retrocompatible con los 4 servicios que hoy
consume el panel de super-admin en `pos-heladeria`, e integración de Sentry (ya presente como
dependencia pero sin usar) que solo reporta en producción y solo fallas técnicas inesperadas —
reutilizando en todo lo posible la infraestructura compartida ya existente
(`get_or_404`/`ensure_unique` en `app/core/crud.py`, `validate_billing_cycle_price` en
`app/core/plan_limits.py`, `get_current_super_admin`/`get_valid_token_data` en
`app/core/dependencies.py`) sin modificarla, porque la usan otros 20+ módulos.

## Technical Context

**Language/Version**: Python 3.12 (`pos-backend/Dockerfile`)

**Primary Dependencies**: FastAPI 0.136.3, Starlette 1.2.0 (middleware ASGI), Pydantic 2.13.4 /
pydantic-settings 2.14.1, SQLAlchemy 2.0.50, sentry-sdk 2.61.0 (**ya declarado en
`requirements.txt`, instalado pero sin ningún uso en `app/` hoy** — no se necesita justificar
como dependencia nueva, solo integrarlo)

**Storage**: PostgreSQL 16 (schema-per-tenant; el módulo super-admin opera sobre el schema
compartido `shared` vía `get_shared_db`/`with_db(None)`) — sin cambios de esquema en este spec

**Testing**: `unittest` de la librería estándar (`python -m unittest app.characterization_tests....`,
patrón ya usado por `test_super_admin_plans.py` y `test_super_admin_payment_catalog.py`), más
`starlette.testclient.TestClient` — **corrección tras auditar el uso real**: `TestClient` no se
usa hoy en ningún test del repo (`grep -rn "TestClient(" app` → 0 resultados; los tres archivos
de `test_invitations_*.py` mencionan en su docstring que deliberadamente **no** lo usan, invocan
las funciones del router en proceso, igual que super-admin). Es, por tanto, un patrón nuevo para
este repositorio — necesario porque es la única forma de ejercer el middleware/los
`exception_handler` ASGI reales (research.md § 5/§ 7); no requiere ninguna dependencia nueva
(`httpx==0.28.1` ya está en `requirements.txt`, requerido por `TestClient`)

**Target Platform**: Servidor Linux (Docker), backend HTTP servido con uvicorn

**Project Type**: Web service (API FastAPI) — este spec toca únicamente el backend

**Performance Goals**: Sin objetivo de performance nuevo; el middleware añadido debe agregar una
sobrecarga despreciable (un `try/except` y, como mucho, una generación de UUID) sobre las
solicitudes al prefijo `/api/v1/super-admin`, y ningún costo sobre el resto de rutas

**Constraints**: Cambios confinados a `app/api/v1/super_admin/**`, la infraestructura mínima y
reutilizable de manejo de errores que ese módulo necesita (nuevos módulos en `app/core/`), y el
cableado de arranque en `app/main.py`/`app/core/config.py`; ninguna función ya usada por otros
módulos (`app/core/crud.py`, `app/core/plan_limits.py`, `app/core/dependencies.py` salvo
`get_current_super_admin`, que es exclusiva de este módulo) cambia de firma ni de comportamiento

**Scale/Scope**: 3 archivos de router (`router.py`, `plans_router.py`, `payment_methods_router.py`,
529 líneas en total) y sus esquemas; ~9 endpoints

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

Evaluado contra los trece principios de `.specify/memory/constitution.md` (v3.0.0):

- **I. Nace de un spec** — ✅ Cumple: este plan deriva de `spec.md` (feature 068), aprobado antes
  de tocar código.
- **II. Comportamiento existente protegido** — ✅ Cumple por diseño: el middleware de traducción
  HTTP se activa solo para el prefijo `/api/v1/super-admin`; ninguna regla de negocio, código de
  estado HTTP ni mensaje de error existente cambia de valor (FR-014). El único cambio de forma de
  respuesta (agregar `success`/`error`/`request_id` junto al `detail` ya existente) es aditivo y
  fue una decisión explícita registrada en `spec.md` § Clarifications, no una migración
  silenciosa.
- **III. Characterization tests protegen lo heredado** — ✅ Cumple, verificado en la auditoría de
  Fase 0: `test_super_admin_plans.py` y `test_super_admin_payment_catalog.py` invocan las
  funciones del router **directamente** (sin pasar por la app ASGI ni por lo tanto por el nuevo
  middleware) y aseveran `HTTPException`/`status_code` sobre `ensure_unique`/`get_or_404`, que
  este plan **no modifica**. Ningún characterization test existente queda en rojo por este cambio;
  se documenta la evidencia en `research.md`.
- **IV. Nuevos specs pueden introducir comportamiento nuevo** — ✅ Aplica: el envelope de error
  estructurado y el reporte a Sentry en producción son comportamiento nuevo, autorizado por
  `spec.md`.
- **V. Nuevas funcionalidades antes que refactor oportunista** — ✅ Cumple: se descarta
  deliberadamente "domain-ificar" cada punto de `raise` existente en el router (todos ya delegan
  en helpers compartidos); el refactor se limita a lo que el spec exige. Ver decisión en
  `research.md`.
- **VI. Evolución incremental** — ✅ Cumple: un solo tipo de cambio (manejo de errores/observabilidad)
  en un solo módulo; no se mezcla con cambios de arquitectura de negocio, migración de datos, ni
  con el frontend.
- **VII. Compatibilidad con datos históricos** — N/A: el módulo no toca facturas ni datos
  históricos de negocio.
- **VIII. Evolución del modelo de datos** — N/A: no hay cambios de esquema/migraciones en este
  spec.
- **IX. Dependencias nuevas justificadas** — ✅ N/A en la práctica: `sentry-sdk` ya está en
  `requirements.txt`; no se agrega ninguna dependencia nueva.
- **X. Verificación obligatoria** — ✅ Se cubre en Fase 1/`quickstart.md` con characterization
  tests existentes (deben seguir en verde) + tests nuevos vía `TestClient` para el envelope y el
  middleware.
- **XI. Decisiones de negocio vs. técnicas** — ✅ La única decisión con matiz de negocio
  (compatibilidad con el frontend existente) ya quedó registrada explícitamente en `spec.md` §
  Clarifications, con quién la tomó (el usuario, en la sesión de `/speckit-specify`) y cuándo.
- **XII. Trazabilidad** — ✅ Cadena completa: Necesidad (pedido del usuario) → Spec (068) → Plan
  (este documento, con research.md como evidencia) → Implementación (tasks.md, pendiente) → Tests
  → Verificación.
- **XIII. Todo en español de Colombia** — ✅ Este plan y sus artefactos se escriben en español.

**Resultado**: PASS. No hay violaciones que requieran `Complexity Tracking`.

## Project Structure

### Documentation (this feature)

```text
specs/068-manejo-excepciones-super-admin/
├── plan.md              # This file (/speckit-plan command output)
├── research.md          # Phase 0 output (/speckit-plan command)
├── data-model.md         # Phase 1 output (/speckit-plan command)
├── quickstart.md        # Phase 1 output (/speckit-plan command)
├── contracts/            # Phase 1 output (/speckit-plan command)
│   └── error-envelope.md
└── tasks.md             # Phase 2 output (/speckit-tasks command - NOT created by /speckit-plan)
```

### Source Code (repository `../pos-backend`)

```text
pos-backend/
├── app/
│   ├── main.py                          # MODIFICADO: registra el nuevo middleware
│   │                                     # (alcance /api/v1/super-admin) e inicializa
│   │                                     # sentry_sdk (gate: ENVIRONMENT == "prod")
│   ├── core/
│   │   ├── config.py                    # MODIFICADO: + SENTRY_DSN (Optional)
│   │   ├── crud.py                      # SIN CAMBIOS (usado por 24 archivos)
│   │   ├── plan_limits.py               # SIN CAMBIOS (usado por 19+ archivos)
│   │   ├── dependencies.py              # SIN CAMBIOS
│   │   ├── exceptions.py                # SIN CAMBIOS (excepciones de auth existentes)
│   │   ├── domain_errors.py             # NUEVO: jerarquía de excepciones de dominio,
│   │   │                                # sin import de FastAPI/Starlette
│   │   ├── error_response.py            # NUEVO: construcción del envelope
│   │   │                                # {success, error, request_id, detail} +
│   │   │                                # mapeo status HTTP / código estable
│   │   └── error_middleware.py          # NUEVO: middleware ASGI reutilizable,
│   │                                     # parametrizado por prefijo de ruta;
│   │                                     # traduce domain_errors + HTTPException +
│   │                                     # Exception inesperada; captura a Sentry
│   │                                     # (solo prod, solo inesperadas)
│   └── api/v1/super_admin/
│       ├── router.py                     # Posiblemente modificado: casos puntuales
│       │                                  # que hoy no tienen characterization test
│       │                                  # protegiéndolos podrán usar domain_errors
│       │                                  # directamente en vez de HTTPException
│       ├── plans_router.py               # Sin cambios de comportamiento (sigue
│       │                                  # usando get_or_404/ensure_unique)
│       └── payment_methods_router.py     # Ídem
└── app/characterization_tests/
    ├── test_super_admin_plans.py         # SIN CAMBIOS (deben seguir en verde)
    ├── test_super_admin_payment_catalog.py  # SIN CAMBIOS (deben seguir en verde)
    └── test_super_admin_error_envelope.py   # NUEVO (vía TestClient): valida el
                                              # envelope de error end-to-end
```

**Structure Decision**: Proyecto único (backend FastAPI, sin frontend en este spec). Toda la
infraestructura verdaderamente nueva y reutilizable vive en `app/core/` bajo nombres propios
(`domain_errors.py`, `error_response.py`, `error_middleware.py`) para no mezclarse con
`app/core/exceptions.py` (excepciones de autenticación ya existentes, usadas fuera de
super-admin) ni con `app/core/crud.py`/`app/core/plan_limits.py` (helpers compartidos que este
spec no modifica). La activación queda confinada al módulo super-admin registrando el middleware
con `path_prefix="/api/v1/super-admin"` en `app/main.py`; migrar otro módulo en el futuro es
agregar su prefijo a esa misma lista, sin duplicar la infraestructura.

## Complexity Tracking

*Sin violaciones de la constitución que requieran justificación — tabla omitida.*
