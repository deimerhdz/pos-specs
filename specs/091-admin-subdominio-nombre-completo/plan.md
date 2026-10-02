# Implementation Plan: Acceso de Super Admin en `admin.skeilopos.com` y Nombre Completo en Invitaciones

**Branch**: `091-admin-subdominio-nombre-completo` (rama de la spec en `pos-specs`; las ramas de código de `pos-backend` y `pos-heladeria` siguen el Principio XIV, ver abajo) | **Date**: 2026-10-02 | **Spec**: [spec.md](./spec.md)

**Input**: Feature specification from `/specs/091-admin-subdominio-nombre-completo/spec.md`

## Summary

La spec tiene dos mitades independientes: (A) aislar el acceso de plataforma en `admin.skeilopos.com` y (B) capturar y mostrar el nombre completo al invitar usuarios. Todo lo siguiente sale de leer `pos-backend` y `pos-heladeria` (ambos en `develop`, árbol limpio) y de sondas HTTP de solo lectura a producción (detalle y evidencia en [research.md](./research.md)):

1. **El 404 tiene una causa probable en código, no en el hosting.** Hoy `admin.skeilopos.com/login` y sus rutas profundas responden 200 (el relleno de SPA ya existe). Lo que falla es que el frontend **no reconoce `admin` como plataforma**: lo toma por un negocio llamado "admin", manda `X-Tenant-Host: admin`, el backend (que deja entrar al Super Admin cuando no halla negocio) autentica, y desde ahí cada llamada de negocio devuelve **404 "Tenant not found for host 'admin'"**. Como un `curl` no ejecuta el SPA, la **primera tarea** es reproducirlo en navegador y registrar el resultado antes de corregir (D1).
2. **Aislamiento (A)**: el backend reconoce la plataforma **solo** por el host explícito `admin` (el frontend lo manda en contexto Super Admin); sin cabecera, con host desconocido o con el dominio raíz **no busca ninguna cuenta** y responde el mismo 401 de siempre. Los negocios existentes no cambian (FR-010). La reserva de `admin`, `assets`, `api` y `docs` (más `www`, `app`) vive en **una** constante del backend, validada en el alta de negocio, y se refleja en el frontend (formulario y resolver). El resolver del frontend pasa a tres estados (plataforma / negocio / no reconocido) para que el dominio raíz no ofrezca login (FR-006).
3. **Nombre completo (B)**: columna nueva y opcional `user_invitations.name`; validación única (obligatorio → longitud 2–100 → lista blanca de letras latinas, espacio, apóstrofe, guion, ≥ 2 letras) implementada en backend y frontend con los **mismos vectores de prueba**; saludo escapado en el correo; la cuenta nace con ese nombre al aceptar; lista de usuarios y de invitaciones pendientes muestran nombre e inicial, con el correo como respaldo para cuentas anteriores; el formulario se limpia (campos y errores) al cancelar.
4. **Hallazgo que corrige un supuesto de la spec (H-10)**: el backend **no tiene** `POST`/`PATCH /super-admin/users`; el formulario del Super Admin llama a endpoints inexistentes (405 verificado). La Historia 6 se planifica como **solo frontend** (validador + presentación) y se aísla en su propio incremento; la decisión de fondo queda **abierta (O-1)** con recomendación en [research.md](./research.md) D11.
5. **Sin dependencias nuevas.** Una migración aditiva y reversible. Ninguna factura se toca.

## Technical Context

**Language/Version**: `pos-backend`: Python 3.12 (FastAPI, Pydantic v2, SQLAlchemy 2, Alembic). `pos-heladeria`: Angular 21 / TypeScript (signals, formularios reactivos, Tailwind).

**Primary Dependencies**: ninguna nueva (Principio IX). Backend: `html` y `unicodedata` de la biblioteca estándar. Frontend: `String.prototype.normalize` y expresiones regulares nativas.

**Storage**: PostgreSQL 16, schema `shared`. **Un cambio**: `shared.user_invitations.name VARCHAR(100) NULL`. Ver [data-model.md](./data-model.md).

**Testing**: backend con `unittest` sobre SQLite en memoria (`auth_fixtures`, mismo patrón de las specs 031/037): `python -m unittest app.characterization_tests.<módulo> -v`; frontend con `ng test` (Vitest + jsdom). Verificación manual en navegador sobre `admin.localhost:4200`, `localhost:4200` y `<negocio>.localhost:4200` y comprobación de producción solo-lectura tras desplegar ([quickstart.md](./quickstart.md)).

**Target Platform**: API en Linux/Docker detrás de Cloudflare (`api.skeilopos.com`); SPA servido por Cloudflare en `<host>.skeilopos.com`. El enlace de `admin.skeilopos.com` al despliegue del SPA es trabajo de despliegue (la spec lo declara dependencia, y hoy ya responde).

**Project Type**: web — `pos-backend` + `pos-heladeria`; la spec vive en `pos-specs`.

**Performance Goals**: sin cambio medible. El login añade una comparación de cadena antes de la consulta existente; el rechazo temprano evita una consulta.

**Constraints**:
- **FR-010 / SC-003**: el login de un negocio existente en su subdominio es idéntico (misma consulta, misma consumición de invitación, mismo token).
- **Indistinguibilidad** (FR-003/004/005/006): todo rechazo por aislamiento es el mismo `401 Invalid credentials`; los motivos reales solo al log.
- **Paridad backend/frontend** de la lista de reservados y de las reglas del nombre: pruebas con vectores compartidos en ambos repos.
- **Datos intactos**: sin backfill; invitaciones y cuentas anteriores siguen funcionando (FR-022, FR-024).
- **Compatibilidad de despliegue**: ver [research.md](./research.md) D12 (orden y ventana de 422 con SPA antiguo).

**Scale/Scope**:
- `pos-backend`: `app/core/reserved_hosts.py` y `app/core/person_name.py` (**nuevos**), `auth/routes.py` (login + consumo), `super_admin/schemas.py` (host reservado), `invitations/schemas.py` + `router.py`, `core/models.py` (columna), `core/mail.py` (saludo), 1 migración Alembic; tests nuevos y ampliados.
- `pos-heladeria`: núcleo de `core/tenant/` (modelo, resolver, servicio, inicializador, guard), interceptor, login, formulario de negocio, módulo `users` (formulario, servicio, interfaces, página, lista de pendientes), módulo `super-admin` (formulario y página de usuarios), 2 helpers nuevos en `shared/`; ambos `environment*.ts`.
- `pos-specs`: anomalías A-99…A-102 (**antes de implementar**), `implementation-notes.md` (diagnóstico y verificación).
- 6 historias en **3 incrementos** (abajo).

### Incrementos (Principio VI)

| Incremento | Historias | Qué entrega | Se puede desplegar solo |
|---|---|---|---|
| **I1 — Acceso y aislamiento** | 1, 2 | login por host explícito; reservados; contexto de 3 estados en el SPA; formulario de negocio con reservados | Sí |
| **I2 — Nombre completo en invitaciones** | 3, 4, 5 | migración, validación, correo, formulario, listas | Sí (tras la migración) |
| **I3 — Formulario del Super Admin** | 6 | validador y presentación en la pantalla del Super Admin (solo frontend; decisión O-1) | Sí, depende de los helpers de I2 |

## Constitution Check

*GATE: debe pasar antes de la Fase 0. Re-evaluado tras la Fase 1 (final de la sección).*

| Principio | Evaluación |
|---|---|
| I. Las Nuevas Funcionalidades Nacen de un Spec | ✅ Pass — `spec.md` con clarificaciones (2026-10-02), checklist de calidad, 25 FR y trazabilidad a los 13 escenarios del brief. |
| II. El Comportamiento Existente Sigue Protegido | ✅ Pass **condicionado a registrar antes de implementar**: cambian tres comportamientos existentes por decisión de negocio (login sin cabecera/host desconocido ya no entra como Super Admin; invitar exige nombre; nuevos reservados). Se registran como **A-99, A-100, A-101** (+ **A-102** del hallazgo H-10) en `registro-de-anomalias.md` **antes** de tocar código — es la primera tarea de `tasks.md`. El resto se conserva: login de negocios (FR-010), consumo de invitación, contraseña temporal, rutas de comensal. |
| III. Los Characterization Tests Protegen el Comportamiento Heredado | ✅ Pass — verificado: ningún test `"CONGELA comportamiento actual:"` cubre `login()`, invitaciones ni el alta de negocio (H-12). Los tests de invitaciones (spec 037) pasan a enviar el campo nuevo y `test_auth_login_invitation_consumption` gana casos; **ningún test se debilita**, y los cambios citan la anomalía que los ampara en el mismo commit. Evidencia: suite completa de `pos-backend` en verde. |
| IV. Los Nuevos Specs Pueden Introducir Nuevo Comportamiento | ✅ Pass — comportamiento acotado y medible (SC-001…SC-008). |
| V. Nuevas Funcionalidades Antes que Refactorizaciones Oportunistas | ✅ Pass — no se escapan `tenant_name`/`email`/`password` del correo (deuda aparte), no se crea handler global de 422, no se añade `POST /super-admin/users` (D7, D8, D11), no se toca la lógica de refresco ni de recuperación de contraseña (el `forgot-password` del Super Admin queda como está). |
| VI. Evolución Incremental | ✅ Pass — tres incrementos, cada uno verificable y desplegable por separado (tabla arriba); la migración vive solo en I2; no se mezclan datos, arquitectura y comportamiento en una misma unidad. |
| VII. Compatibilidad con Datos Históricos | ✅ Pass — ninguna factura, venta ni caja se lee o escribe; ninguna fila existente se reescribe. |
| VIII. Evolución del Modelo de Datos | ✅ Pass — [data-model.md](./data-model.md) especifica entidad/campo nuevo, valor por defecto (`NULL`), compatibilidad (invitaciones y cuentas anteriores), estrategia de migración y de **rollback** (`downgrade` borra solo esa columna). |
| IX. Dependencias Nuevas Permitidas con Justificación | ✅ Pass (no aplica) — cero dependencias; la alternativa `regex` (Unicode) se evaluó y descartó (research D5). |
| X. Verificación Obligatoria | ✅ Pass (planificado) — pruebas unitarias y de contrato en ambos repos con vectores compartidos; prueba de login por las filas 1–8 y 10–11 del contrato (host × cuenta); verificación en navegador de los 13 escenarios; comprobación de producción solo-lectura. El diagnóstico real **precede** a la corrección (D1). |
| XI. Decisiones de Negocio Frente a Decisiones Técnicas | ✅ Pass — qué se rechaza y qué reservar es decisión de negocio (ya en la spec); "el host explícito `admin`" y "tres estados en el SPA" son decisiones técnicas del plan. La decisión O-1 (Historia 6) se **eleva** al dueño de la spec en vez de resolverse en silencio. |
| XII. Trazabilidad | ✅ Pass — Necesidad (brief del 2026-10-02) → spec + clarificaciones → A-99…A-102 → este plan → research/data-model/contracts/quickstart → `tasks.md` → tests y verificación. |
| XIII. Todo en Español de Colombia | ✅ Pass — artefactos, comentarios, mensajes de error y textos de pantalla/correo en español de Colombia. Commits y ramas en inglés. |
| XIV. Estrategia y Convención de Ramas | ✅ Pass (planificado) — antes de tocar código, en **cada** repo y **desde `develop`** (la rama actual): `feat/091-platform-host-invitation-name` en `pos-backend` y en `pos-heladeria`. `/speckit-tasks` lo deja como tarea inicial. |
| XV. Política de Commits | ✅ Pass (planificado) — commits pequeños por unidad (módulo de reservados; login; schema de alta; migración+modelo; validador de nombre; invitaciones; correo; resolver; interceptor; formularios; listas; docs), en inglés, Conventional Commits, **sin marcas de IA**, solo cuando el usuario lo pida. |

**Complexity Tracking**: sin violaciones que justificar.

**Re-chequeo post Fase 1**: el diseño no añadió dependencias, variables de entorno ni endpoints. Dos hallazgos quedaron **absorbidos** sin cambiar el veredicto: (1) H-4 muestra que el backend por sí solo no arregla el 404 — el contrato del frontend ([contracts/tenant-context-resolution.md](./contracts/tenant-context-resolution.md)) es parte esencial del incremento I1, no un detalle; (2) H-10 obligó a acotar la Historia 6. **Riesgos abiertos, declarados y con verificación asignada**: ventana de 422 con SPA antiguo en caché (D12); `X-Tenant-Host` es política y no frontera criptográfica (D2); decisión **O-1** pendiente del dueño de la spec.

## Project Structure

### Documentation (this feature)

```text
specs/091-admin-subdominio-nombre-completo/
├── spec.md              # Especificación (con clarificaciones 2026-10-02)
├── plan.md              # Este archivo (/speckit-plan)
├── research.md          # Fase 0: hechos verificados H-1…H-12 + decisiones D1–D13
├── data-model.md        # Fase 1: columna nueva, entidades y reglas de validación
├── quickstart.md        # Fase 1: verificación previa, pruebas, recorrido manual y producción
├── contracts/
│   ├── auth-login-platform-isolation.md   # POST /auth/login: tabla host × cuenta
│   ├── tenant-creation-reserved-hosts.md  # POST /super-admin/tenants: reservados
│   ├── invitations-full-name-api.md       # POST /invitations, lista, reenvío, correo
│   ├── full-name-rules.md                 # reglas del nombre + vectores compartidos
│   └── tenant-context-resolution.md       # SPA: host → contexto → cabecera
├── checklists/
│   └── requirements.md
├── implementation-notes.md   # (se crea al implementar) diagnóstico del 404 y verificación
└── tasks.md             # Fase 2 (/speckit-tasks — NO lo crea /speckit-plan)
```

### Source Code (repository root)

Cambian `../pos-backend` y `../pos-heladeria`.

```text
../pos-backend/
├── app/core/
│   ├── reserved_hosts.py                 # NUEVO — PLATFORM_HOST, RESERVED_SUBDOMAINS, is_reserved_subdomain()
│   ├── person_name.py                    # NUEVO — normalize_full_name() + mensajes (es-CO)
│   ├── models.py                         # UserInvitation.name (nullable)
│   └── mail.py                           # invitation_email_body(name=None): saludo escapado
├── app/api/v1/
│   ├── auth/routes.py                    # login(): ámbito por host explícito; consumo con nombre
│   ├── invitations/{schemas,router}.py   # name en create/response; reenvío conserva nombre
│   └── super_admin/schemas.py            # TenantCreateWithUser.host: no reservado
├── alembic/versions/<rev>_091_invitation_full_name.py   # NUEVO — add_column / drop_column
└── app/characterization_tests/
    ├── test_auth_login_platform_isolation.py   # NUEVO — host × cuenta (filas 1–8 y 10–11 del contrato)
    ├── test_reserved_hosts.py                  # NUEVO — lista, normalización, alta de negocio
    ├── test_person_name.py                     # NUEVO — vectores de full-name-rules.md
    ├── test_invitation_email_greeting.py       # NUEVO — saludo, escape, legacy
    ├── test_invitations_create.py / _list.py / _resend_cancel.py   # ampliados (nombre)
    └── test_auth_login_invitation_consumption.py                  # ampliado (nombre / legacy)

../pos-heladeria/
├── src/environments/environment{,.development}.ts   # reservedSlugs completa + platformSlug
└── src/app/
    ├── core/tenant/
    │   ├── app-environment.interface.ts         # platformSlug
    │   ├── tenant-context.model.ts              # + Unrecognized
    │   ├── tenant-resolver.ts (+ .spec.ts)      # 3 estados
    │   ├── tenant-context.service.ts            # isTenant, tenantHostHeader
    │   ├── tenant.initializer.ts                # pasa platformSlug
    │   └── guards/tenant-domain.guard.ts        # exige TENANT
    ├── core/auth/auth-token.interceptor.ts (+ .spec.ts)   # cabecera por tenantHostHeader()
    ├── modules/auth/pages/login.component.ts (+ .spec.ts) # estado "dirección no válida"
    ├── shared/
    │   ├── validators/full-name.validator.ts (+ .spec.ts)  # NUEVO — mismos vectores
    │   └── person-display.ts (+ .spec.ts)                   # NUEVO — displayName / initialOf
    ├── modules/users/                            # formulario + servicio + interfaces + página + pendientes
    └── modules/super-admin/                      # tenant-form (reservados), admin-user-form, página de usuarios
```

**Structure Decision**: web con dos repositorios afectados. La política de reservados y las reglas del nombre viven en **módulos pequeños y puros** en cada lado (`reserved_hosts.py`/`person_name.py` y `full-name.validator.ts`) para que una sola prueba por repositorio fije su contrato; el helper de presentación va en `shared/` porque lo consumen tres pantallas de dos módulos distintos (a diferencia de la spec 090, aquí la reutilización es real y presente).

## Complexity Tracking

> Sin violaciones de la Constitución que justificar (ver Constitution Check).
