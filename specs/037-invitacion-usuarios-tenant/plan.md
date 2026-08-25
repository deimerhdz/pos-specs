# Implementation Plan: Alta de usuarios internos por invitación

**Branch**: `037-invitacion-usuarios-tenant` | **Date**: 2026-08-25 | **Spec**: [spec.md](./spec.md)

**Input**: Feature specification from `/specs/037-invitacion-usuarios-tenant/spec.md`

## Summary

Hoy un ADMIN de tenant crea usuarios internos con `POST /users`, escribiendo a mano correo y
contraseña (`UserCreate.password`) — un tercero conoce la contraseña de la cuenta que va a usar
otra persona. Esta spec reemplaza ese mecanismo por invitación: el ADMIN solo aporta correo y rol
(FR-001); el sistema genera una contraseña temporal (mismo mecanismo que ya usa `tenant_create()`
para el primer admin de un tenant), la envía por correo, y no crea ningún `User` hasta que la
persona invitada se autentica por primera vez con esa contraseña — momento en el que nace la cuenta
real, con el flujo de cambio obligatorio de contraseña ya existente (spec 001/031) activado.

La pieza técnica central es dónde vive el estado de "invitación pendiente" sin tabla de sesiones ni
mecanismo de tokens firmados: una tabla nueva `UserInvitation` (schema `shared`, mismo patrón que
`PasswordResetToken` de spec 031) con **una fila por correo+tenant** mientras esté vigente, y un
índice único parcial que resuelve a nivel de base de datos la condición de carrera de dos ADMIN
invitando el mismo correo casi simultáneamente. El consumo de la invitación se resuelve extendiendo
`POST /auth/login` — no un endpoint nuevo — con un branch que solo se activa cuando la búsqueda
normal de `User` no encuentra nada, ver [research.md](./research.md) Decisión 7.

## Technical Context

**Language/Version**: Backend — Python 3.14 (venv `pos-backend/env`). Frontend — TypeScript 5.9.2
(Angular 21, standalone components + signals, sin NgModules).

**Primary Dependencies**:
- Backend: FastAPI, SQLAlchemy 2.0 (sync), Alembic, Pydantic 2, `bcrypt` (hash/verify, ya en
  `app/core/utils.py`), `httpx` (envío de correo síncrono vía `send_email()`, ya en uso para
  errores/timeouts). **Ninguna dependencia nueva** (Principio IX no aplica).
- Frontend: Angular 21 (standalone + signals), Reactive Forms, Tailwind CSS 4. Ninguna dependencia
  nueva.

**Storage**: PostgreSQL 16, schema `shared` (no per-tenant) para lo nuevo — tabla
`user_invitations`, vía Alembic con una sola migración (sin `@for_each_tenant_schema`, mismo patrón
que `password_reset_tokens` de spec 031, porque `Tenant`/`Role`/`User` viven en `shared`). Sin
cambios de esquema en `User` ni `Tenant` — ver [data-model.md](./data-model.md).

**Testing**: `unittest` vía `python -m unittest` (sin pytest en el repo). Se extiende
`app/characterization_tests/auth_fixtures.py` (spec 031) con `UserInvitation` y un helper
`make_invitation(...)`, y se agregan módulos de test nuevos por historia de usuario, llamando las
funciones de router directamente (sin `TestClient`, sin precedente de eso en el repo). Frontend:
sin `.spec.ts` propio para los componentes nuevos/modificados de este módulo (mismo criterio que
`UsersPageComponent` hoy) — validación manual vía `ng serve` (ver [quickstart.md](./quickstart.md)).

**Target Platform**: Linux server (`pos-backend` en producción) + navegador (`pos-heladeria`).

**Project Type**: Web application (backend FastAPI + frontend Angular, dos repositorios
independientes, siblings de este repo `pos-specs`).

**Performance Goals**: Sin objetivo de throughput nuevo. Única particularidad: el envío de correo
de invitación es **síncrono** (no Celery, a diferencia del resto del backend) — bloquea la
respuesta HTTP hasta el `timeout=10.0` de `send_email()` en el peor caso, aceptable porque invitar
es una acción administrativa de baja frecuencia sin requisito de latencia en el spec (ver
[research.md](./research.md) Decisión 4, que también justifica por qué esta spec se aparta
deliberadamente del patrón fire-and-forget usado en specs anteriores).

**Constraints**:
- FR-004 exige eliminar `POST /users` (`UserCreate` con contraseña) por completo, sin vía alterna.
- FR-006/FR-012 exigen que la contraseña temporal nunca se exponga y que un fallo de envío deje la
  invitación inutilizable con error explícito — impulsa la Decisión 4 (envío síncrono).
- El límite de usuarios del plan vigente de un tenant (spec 033, FR-005/006/007, comportamiento
  protegido por el Principio II) no puede quedar eludible por esta spec — impulsa la Decisión 5
  (contar invitaciones `pending` contra ese límite).
- No se modifica el flujo de cambio obligatorio de contraseña tras el primer login (FR-008) ni
  ningún endpoint de spec 031 (`forgot-password`, `reset-password`, `change-password`).
- Fuera de alcance explícito de `spec.md`: registro de auditoría de acciones sobre invitaciones,
  límite de tasa de invitaciones por tenant, edición de correo de una invitación existente.

**Scale/Scope**: 1 tabla nueva (`user_invitations`), 4 endpoints nuevos (`POST /invitations`,
`GET /invitations`, `POST /invitations/{id}/resend`, `POST /invitations/{id}/cancel`) + 1
modificado (`POST /auth/login`) + 1 eliminado (`POST /users`) en `pos-backend`; en `pos-heladeria`:
1 formulario reemplazado (invitación en vez de creación con contraseña), 1 listado extendido con una
sección de pendientes, 2 acciones nuevas (reenviar/cancelar).

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| Principio | Evaluación | Estado |
|---|---|---|
| **I. Las Nuevas Funcionalidades Nacen de un Spec** | `spec.md` existe, aprobado, con 4 historias priorizadas, 19 FRs y 2 clarificaciones ya resueltas (sesión 2026-08-25) antes de este plan. | PASS |
| **II. El Comportamiento Existente Sigue Protegido** | Cambio de comportamiento explícito y autorizado por la propia `spec.md`/Assumptions: se elimina por completo el mecanismo de creación de usuario con contraseña directa (`POST /users`), "decisión confirmada con el usuario", sin vía alterna. Todo lo demás es aditivo: `GET/PATCH /users/*` no cambian; el flujo de cambio obligatorio de contraseña (spec 001/031) no se modifica (FR-008, explícito). El único comportamiento protegido que este plan toca de cerca sin spec propia es el límite de usuarios del plan (spec 033) — no se debilita, se **extiende** su conteo para que la nueva vía de creación (invitación) no lo eluda (research.md Decisión 5); es una consecuencia técnica necesaria para no romper una garantía ya protegida, no un cambio de esa regla de negocio. | PASS |
| **III. Los Characterization Tests Protegen el Comportamiento Heredado** | Confirmado: no existe ningún test `"CONGELA comportamiento actual:"` para `app/api/v1/users` (módulo donde se elimina `create_user`) — nada protegido que este plan ponga en rojo sin autorización. `login()` (spec 001, sin tests dedicados hoy) se extiende de forma aditiva: la rama nueva solo se activa cuando `user is None`, así que cualquier test futuro de login con usuario existente sigue viendo el mismo camino de código de siempre. Los tests de `enforce_plan_limit` de spec 033 (si existen para otros recursos) no deberían romperse: solo se modifica el conteo de la rama `"usuarios"`. | PASS |
| **IV. Los Nuevos Specs Pueden Introducir Nuevo Comportamiento** | Todo el flujo de invitación es comportamiento nuevo — no se exige equivalencia con el pasado, solo conformidad con `spec.md` y que `POST /users` desaparezca sin dejar rastro (FR-004). | PASS |
| **V. Nuevas Funcionalidades Antes que Refactorizaciones Oportunistas** | No se fusiona el listado de invitaciones dentro de `GET /users`/`UserResponse` pese a ser tentador (research.md Decisión 13); no se generaliza `PasswordResetToken` para reusarlo como motor de invitaciones (research.md Decisión 2); no se agrega un campo de nombre al formulario de invitación pese a que `User.name` es `NOT NULL` — se resuelve técnicamente (Decisión 8) en vez de expandir el alcance de la UI. | PASS |
| **VI. Evolución Incremental** | El alcance se divide en las mismas unidades que las historias del spec (US1 crear → US2 consumir → US3 listar → US4 reenviar/cancelar), cada una con su propio archivo de test. La eliminación de `POST /users` viaja en el mismo incremento por ser la otra mitad indivisible de FR-004 (no tiene sentido tener ambos mecanismos activos a la vez), no una refactorización aparte. | PASS |
| **VII. Compatibilidad con Datos Históricos** | No se toca `Sale`/`Payment`/`SaleInvoice` ni ninguna venta o factura ya emitida — esta spec vive enteramente en `auth`/`users`/`invitations` (schema `shared`). | PASS |
| **VIII. Evolución del Modelo de Datos** | Ver data-model.md: tabla nueva `user_invitations` (rollback `op.drop_table`, sin datos preexistentes que preservar) en `shared`, sin `@for_each_tenant_schema`. Sin columnas nuevas en `User`/`Tenant`. Estrategia de rollback explícita en research.md, sección final. | PASS |
| **IX. Dependencias Nuevas Permitidas con Justificación** | No se añade ninguna dependencia (Technical Context). | PASS (no aplica) |
| **X. Verificación Obligatoria** | Cada historia de usuario tiene su "Independent Test" en `spec.md`; quickstart.md los traduce a comandos `unittest` ejecutables, incluida una verificación explícita de que el conteo de plan (spec 033) y el resto de `login()` (spec 001/031) siguen intactos. | PASS |
| **XI. Decisiones de Negocio Frente a Decisiones Técnicas** | Las decisiones de negocio (eliminar creación con contraseña, invitación por correo+rol, sin límite de tasa, sin auditoría en esta versión) están en `spec.md`. *Cómo* modelar la invitación (una fila vs. histórico de tokens), *cómo* garantizar unicidad bajo concurrencia (índice parcial), *cómo* enviar el correo de forma que un fallo sea visible (síncrono) y *cómo* no eludir el límite de plan (conteo extendido) son decisiones técnicas documentadas aparte en research.md. | PASS |
| **XII. Trazabilidad** | Cadena completa: `spec.md` (Necesidad+Spec+Decisión, sesión de clarificación 2026-08-25) → este `plan.md`/`research.md` (Decisión técnica) → `tasks.md` (Fase 2, no generada por este comando) → implementación → tests nuevos de `invitations`/`auth` + verificación explícita de no-regresión sobre spec 033/031 → `quickstart.md` (Verificación). | PASS |
| **XIII. Todo en Español de Colombia** | Este plan y todos sus artefactos (research.md, data-model.md, contracts/, quickstart.md) se escriben en español de Colombia, igual que `spec.md`. | PASS |

Sin violaciones. La tabla de Complexity Tracking al final de este documento queda vacía.

## Project Structure

### Documentation (this feature)

```text
specs/037-invitacion-usuarios-tenant/
├── plan.md              # Este fichero (/speckit-plan)
├── research.md          # Fase 0 (/speckit-plan) — decisiones técnicas y alternativas descartadas
├── data-model.md         # Fase 1 (/speckit-plan) — entidades, columnas, transiciones, migraciones
├── quickstart.md          # Fase 1 (/speckit-plan) — validación ejecutable por historia de usuario
├── contracts/              # Fase 1 (/speckit-plan) — contratos HTTP nuevos/modificados/eliminados
│   ├── invitations-create.md
│   ├── invitations-list.md
│   ├── invitations-resend.md
│   ├── invitations-cancel.md
│   ├── auth-login.md
│   └── users-create-removed.md
└── tasks.md                # Fase 2 (/speckit-tasks — NO se crea en este comando)
```

### Source Code (repositorios sibling de `pos-specs`)

Esta spec vive en `pos-specs`, pero el código que describe está en `../pos-backend` y
`../pos-heladeria` (Constitución §Alcance). Rutas relativas a la raíz de cada repo.

```text
# pos-backend
app/
├── core/
│   ├── models.py                # MODIFICADO — UserInvitation nueva (schema shared); User sin
│   │                               cambios de columnas
│   ├── plan_limits.py            # MODIFICADO — RESOURCE_CONFIG["usuarios"] cuenta también
│   │                               UserInvitation.status='pending' (research.md Decisión 5)
│   ├── mail.py                    # MODIFICADO — invitation_email_body(), mismo estilo que
│   │                                 welcome_email_body
│   └── utils.py                    # SIN CAMBIOS — generate_random_password/generate_passwd_hash
│                                      se reusan tal cual
│
├── api/v1/auth/
│   └── routes.py                    # MODIFICADO — login() gana el branch de consumo de
│                                       invitación cuando user is None (research.md Decisión 7)
│
├── api/v1/users/
│   ├── router.py                     # MODIFICADO — create_user() eliminado; GET/PATCH sin cambios
│   └── schemas.py                     # MODIFICADO — UserCreate eliminado
│
├── api/v1/invitations/                # NUEVO — paquete del router de invitaciones
│   ├── __init__.py
│   ├── router.py                       # POST /invitations, GET /invitations,
│   │                                      POST /invitations/{id}/resend,
│   │                                      POST /invitations/{id}/cancel
│   └── schemas.py                       # InvitationCreate, InvitationResponse
│
├── main.py                              # MODIFICADO — registra invitations_router
│
├── characterization_tests/
│   ├── auth_fixtures.py                  # MODIFICADO — UserInvitation en _TABLE_NAMES +
│   │                                        make_invitation(...)
│   ├── test_invitations_create.py         # NUEVO — US1 (FR-001 a FR-006, FR-013 a FR-019)
│   ├── test_invitations_list.py            # NUEVO — US3 (FR-009)
│   ├── test_invitations_resend_cancel.py    # NUEVO — US4 (FR-010, FR-011, FR-012)
│   └── test_auth_login_invitation_consumption.py  # NUEVO — US2 (FR-007, FR-008 sin modificar)
│
└── alembic/versions/
    └── {rev}_user_invitations.py          # NUEVO — tabla user_invitations, schema shared,
                                              sin @for_each_tenant_schema

# pos-heladeria
src/app/modules/users/
├── interfaces/
│   └── user-profile.interface.ts     # MODIFICADO — se elimina password de TenantUserForm/
│                                        TenantUserCreatePayload; se agregan InvitationForm,
│                                        InvitationCreatePayload, PendingInvitation
├── services/
│   └── users.service.ts               # MODIFICADO — createUser() reemplazado por
│                                         invitations.service.ts (ver abajo); loadUsers() sin
│                                         cambios
├── services/
│   └── invitations.service.ts          # NUEVO — loadPendingInvitations(), createInvitation(),
│                                          resendInvitation(), cancelInvitation()
├── components/
│   ├── user-form.component.ts          # REEMPLAZADO por invitation-form.component.ts —
│   │                                      formulario con exactamente correo + rol, sin
│   │                                      contraseña/nombre/teléfono (FR-001)
│   ├── invitation-form.component.ts     # NUEVO
│   ├── user-role-modal.component.ts      # SIN CAMBIOS
│   └── pending-invitations-list.component.ts  # NUEVO — sección de pendientes con
│                                                 reenviar/cancelar (FR-009, FR-010, FR-011)
└── pages/
    └── users-page.component.ts           # MODIFICADO — usa InvitationFormComponent en vez de
                                             UserFormComponent; agrega
                                             PendingInvitationsListComponent como segunda sección
```

**Structure Decision**: cada historia de usuario del spec se mapea a un subconjunto disjunto de los
ficheros de arriba (US1 → `api/v1/invitations/` + `invitation-form.component.ts`; US2 →
`auth/routes.py::login` modificado, sin pantalla propia; US3 →
`pending-invitations-list.component.ts` + `GET /invitations`; US4 → los dos endpoints de acción +
los botones de esa misma lista). No se crea ningún módulo de servicio nuevo en `pos-backend` más
allá del router `invitations` — se sigue el mismo patrón sin capa `service.py` que ya usa `users`
(Principio V). En `pos-heladeria` se reemplaza `UserFormComponent` en vez de modificarlo in-place,
porque su forma cambia por completo (de 5 campos a 2); `UsersPageComponent` se extiende, no se
reescribe, para no arrastrar la paginación/skeleton/roleModal ya existentes de la sección de
usuarios activos.

## Complexity Tracking

> **Fill ONLY if Constitution Check has violations that must be justified**

Sin violaciones — tabla vacía.
