# Implementation Plan: Rol Mesero con Acceso Restringido a Terminal de Mesas y Órdenes

**Branch**: `075-rol-mesero` | **Date**: 2026-09-04 | **Spec**: [spec.md](./spec.md)

**Input**: Feature specification from `/specs/075-rol-mesero/spec.md`

**Note**: This template is filled in by the `/speckit-plan` command; its definition describes the execution workflow.

## Summary

Agregar un tercer rol asignable desde el panel de administración del tenant,
"Mesero", restringido —tanto en la interfaz como con bloqueo real del lado
del servidor— a exactamente dos capacidades: usar la Terminal de Mesas con
la misma funcionalidad completa que hoy tiene Cajero (tomar pedidos,
gestionar sesiones de mesa, cobrar) y consultar el listado/detalle de
Órdenes en modo solo lectura. Todo lo demás (Caja, Ventas, Inventario,
Reportes, Productos, Usuarios, Ajustes, etc.) queda bloqueado. Admin y
Cajero no cambian.

El enfoque técnico: (1) agregar `MESERO` al catálogo de roles existente
(`shared.roles`) vía una migración de datos idempotente, porque la
inicialización automática de roles solo corre en bases de datos nuevas; (2)
centralizar la verificación de alcance por rol dentro de `get_current_user()`
en el backend —el único punto por el que pasan prácticamente todos los
endpoints de negocio— en vez de tocar cada endpoint individualmente, con una
lista explícita de rutas permitidas (default-deny) documentada como
contrato; (3) extender en el frontend los mecanismos de `roleGuard` y
`NAV_ITEMS` que ya existen y ya restringen a Cajero, sin crear ningún
mecanismo nuevo ahí.

## Technical Context

**Language/Version**: Backend: Python 3.14 (FastAPI 0.136.3). Frontend: TypeScript ~5.9.2 (Angular 21.1)

**Primary Dependencies**: Backend: FastAPI, SQLAlchemy, Alembic, Pydantic (sin dependencias nuevas). Frontend: Angular Router, RxJS (sin dependencias nuevas)

**Storage**: PostgreSQL 16, arquitectura schema-per-tenant con un schema `shared` para roles/usuarios/tenants — el rol Mesero es una fila nueva en `shared.roles`, sin cambio de esquema

**Testing**: Backend: `unittest` (convención de `app/characterization_tests/`, ejecutado con `python -m unittest app.characterization_tests.<modulo> -v`). Frontend: Vitest (unitario/componentes), Playwright (e2e) — ambos ya configurados en `package.json`

**Target Platform**: Backend: servidor Linux (API). Frontend: SPA de navegador, panel de administración del tenant

**Project Type**: Aplicación web (backend + frontend en dos repos separados: `pos-backend`, `pos-heladeria`; este repo, `pos-specs`, solo contiene la documentación)

**Performance Goals**: Ninguno nuevo — la verificación de alcance por rol es una comparación en memoria contra una lista estática, sin impacto medible sobre el resto de endpoints (que no la ejecutan, al aplicar solo cuando `role.name == "MESERO"`)

**Constraints**: No modificar el comportamiento actual de Admin ni Cajero (FR-008); el bloqueo debe ser real del lado del servidor, no solo ocultamiento en la interfaz (FR-007); un cambio de rol debe aplicar de inmediato sin requerir cierre de sesión (FR-009)

**Scale/Scope**: Alcance pequeño y aditivo — un valor de catálogo nuevo, un mecanismo de autorización centralizado en un archivo backend, y extensiones de configuración (no de mecanismo) en cuatro archivos del frontend

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| Principio | Evaluación |
|---|---|
| I. Nace de un spec | ✅ `specs/075-rol-mesero/spec.md`, con las tres decisiones de negocio críticas ya resueltas en su sección Clarifications |
| II. Comportamiento existente protegido | ✅ FR-008/SC-004 exigen explícitamente que Admin y Cajero no cambien; ningún endpoint ni ruta que ya usan pierde ni gana comportamiento — la verificación nueva solo se activa para `role.name == "MESERO"` |
| III. Characterization tests protegen lo heredado | ✅ Ningún test `"CONGELA comportamiento actual:"` se toca; la verificación nueva es aditiva y no reemplaza ninguna lógica existente de `get_current_user()`/`require_tenant_admin` |
| IV. Los nuevos specs pueden introducir comportamiento nuevo | ✅ Aplica — el bloqueo real por rol para Mesero es comportamiento nuevo, autorizado explícitamente por el spec |
| V. Nueva funcionalidad antes que refactor oportunista | ✅ No se toca nada fuera de lo que esta funcionalidad requiere (no se convierte `Role.name` en enum de base de datos, no se reordena código no relacionado) |
| VI. Evolución incremental | ✅ La migración de datos, el mecanismo de autorización backend, y los cambios de frontend son todos partes inseparables de una misma capacidad (el rol Mesero no funciona sin las tres) — no es una mezcla de cambios no relacionados |
| VII. Compatibilidad con datos históricos | N/A — no se toca ninguna factura ni dato histórico |
| VIII. Evolución del modelo de datos | ✅ Ver data-model.md: entidad, valor por defecto, compatibilidad, migración y rollback documentados explícitamente para la fila `MESERO` en `shared.roles` |
| IX. Dependencias nuevas | N/A — no se agrega ninguna dependencia de terceros |
| X. Verificación obligatoria | ✅ quickstart.md define la validación de extremo a extremo; tasks.md (siguiente comando) deberá incluir tests de characterization/unitarios para la verificación de alcance y para las rutas del frontend |
| XI. Decisiones de negocio vs técnicas | ✅ Las tres decisiones de negocio ya están registradas en spec.md; este plan solo toma decisiones técnicas (research.md D1-D6) |
| XII. Trazabilidad | ✅ research.md, data-model.md y los contratos referencian los FR/decisiones de spec.md por número |
| XIII. Español de Colombia | ✅ Todos los artefactos de esta funcionalidad están en español |

**Resultado**: sin violaciones. No se requiere la tabla de Complexity Tracking.

## Project Structure

### Documentation (this feature)

```text
specs/075-rol-mesero/
├── plan.md                          # This file (/speckit-plan command output)
├── research.md                      # Phase 0 output (/speckit-plan command)
├── data-model.md                    # Phase 1 output (/speckit-plan command)
├── quickstart.md                    # Phase 1 output (/speckit-plan command)
├── contracts/
│   ├── backend-endpoint-access.md   # Phase 1 output (/speckit-plan command)
│   └── frontend-route-access.md     # Phase 1 output (/speckit-plan command)
├── checklists/
│   └── requirements.md
└── tasks.md                         # Phase 2 output (/speckit-tasks command - NOT created by /speckit-plan)
```

### Source Code (repository root)

Este repo (`pos-specs`) solo contiene la documentación de la funcionalidad.
El código vive en dos repos hermanos, ya existentes — no se crea ninguna
carpeta ni proyecto nuevo, solo se modifican los archivos concretos
señalados abajo (lista exhaustiva en research.md y en `contracts/`):

```text
../pos-backend/
├── app/core/
│   ├── db.py                              # ROLE_NAMES: agrega "MESERO"
│   └── dependencies.py                    # get_current_user(): verificación de alcance por rol (D2)
├── app/api/v1/users/schemas.py            # RoleName: agrega MESERO
├── alembic/versions/
│   └── <nueva>_seed_mesero_role.py        # INSERT idempotente en shared.roles
└── app/characterization_tests/            # tests nuevos/actualizados para D2

../pos-heladeria/
├── src/app/core/interfaces/user.interface.ts              # UserRole.MESERO
├── src/app/core/guards/role.guard.ts                      # ROLE_DEFAULT_ROUTES[MESERO]
├── src/app/core/config/navigation.config.ts                # NAV_ITEMS: Terminal de mesas y Órdenes
├── src/app/modules/dashboard/routes.ts                     # roleGuard([...]) en 4 rutas
├── src/app/modules/users/interfaces/user-profile.interface.ts  # RoleName type
├── src/app/modules/users/components/invitation-form.component.ts   # <option> Mesero
├── src/app/modules/users/components/user-role-modal.component.ts  # <option> Mesero
└── src/app/modules/users/pages/users-page.component.ts     # ROLE_LABELS / badge
```

**Structure Decision**: Aplicación web con backend y frontend en repos
separados (patrón ya establecido por el resto de `pos-specs`); esta
funcionalidad no introduce ninguna carpeta ni módulo nuevo en ninguno de los
dos repos, solo extiende archivos de configuración/autorización ya
existentes con el tercer rol.

## Complexity Tracking

*Sin violaciones de la Constitution Check — tabla no aplica.*
