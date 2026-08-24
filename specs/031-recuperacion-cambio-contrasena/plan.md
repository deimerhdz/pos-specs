# Implementation Plan: Recuperación y Cambio de Contraseña (Personal)

**Branch**: `031-recuperacion-cambio-contrasena` | **Date**: 2026-08-24 | **Spec**: [spec.md](./spec.md)

**Input**: Feature specification from `/specs/031-recuperacion-cambio-contrasena/spec.md`

## Summary

Spec 001 documentó que hoy **no existe** ningún mecanismo de recuperación de contraseña por correo
para el personal (cajero/admin) — el único camino es `POST /auth/change-password` autenticado
(`RN-AUTH-01`/`RN-AUTH-02`), sin invalidación de sesiones ni aviso por correo. Esta spec agrega dos
flujos: (A) recuperación no autenticada desde el login, con un enlace de un solo uso de 30 minutos
enviado por correo y un límite de 3 solicitudes/15 min en ventana deslizante por correo ingresado;
(B) endurecimiento del cambio autenticado ya existente. Ambos flujos exigen una contraseña de 8-12
caracteres (antes 6-128), distinta de la actual, cierran sesiones de la cuenta al tener éxito
(todas en A, todas-menos-la-de-origen en B) y disparan un correo de aviso.

La pieza técnica central es cómo "cerrar sesiones" sin que el backend tenga hoy una tabla de
sesiones ni una lista positiva de tokens vigentes (solo una blocklist negativa de revocados por
logout): se agrega una columna `tokens_valid_after` en `User` y se reutiliza la relectura de base
de datos que `get_current_user`/`get_authenticated_user`/`refresh-token` **ya hacen en cada
petición** (`RN-AUTH-07`, protegida A-23) para comparar contra el `iat` del JWT — ver
[research.md](./research.md) Decisión 1.

## Technical Context

**Language/Version**: Backend — Python 3.14 (venv `pos-backend/env`). Frontend — TypeScript 5.9.2
(Angular 21, standalone components + signals, sin NgModules).

**Primary Dependencies**:
- Backend: FastAPI, SQLAlchemy 2.0 (sync), Alembic 1.18.4, Pydantic 2, `bcrypt` (hash/verify de
  password, `app/core/utils.py`), `PyJWT` (access/refresh, ya incluye `iat`), `redis.asyncio`
  (blocklist de logout y rate limiting, ambos ya en uso), Celery + `httpx` (envío de correo
  asíncrono, `app/celery_task.py`/`app/core/mail.py`, ya en uso para el correo de bienvenida).
  **Ninguna dependencia nueva** (Principio IX no aplica): `itsdangerous` ya está en
  `requirements.txt` pero se decide **no** usarlo (research.md Decisión 2); `hashlib`/`secrets` son
  librería estándar.
- Frontend: Angular 21 (standalone + signals), Reactive Forms, Tailwind CSS 4. Ninguna dependencia
  nueva.

**Storage**: PostgreSQL 16, schema `shared` (no per-tenant) para todo lo nuevo — columna
`users.tokens_valid_after` y tabla nueva `password_reset_tokens`, ambas vía Alembic con una sola
migración (sin `@for_each_tenant_schema`, mismo patrón que `tenants.logo_url`, porque `User` vive
en `shared`, no en el schema por tenant). Redis (ya en uso) para el límite de solicitudes por
ventana deslizante — ver [data-model.md](./data-model.md).

**Testing**: `unittest` vía `python -m unittest` (sin pytest en el repo). `auth` no tiene tests
hoy (confirmado: no existe `test_auth_*.py`; spec 001 SC-004 ya señalaba esta ausencia) — se agrega
`app/characterization_tests/auth_fixtures.py` (extiende el truco de `schema_translate_map` de
`fixtures.py` para incluir también las tablas del schema `shared`) y módulos de test nuevos por
historia de usuario, llamando las funciones de `routes.py` directamente (sin `TestClient`, sin
precedente de eso en el repo). Frontend: Vitest, extendiendo `auth.service.spec.ts`/
`auth-api.service.spec.ts`; los dos componentes de página nuevos se validan manualmente (mismo
criterio que `LoginComponent`/`ChangePasswordComponent` hoy, sin `.spec.ts` propio).

**Target Platform**: Linux server (`pos-backend` en producción) + navegador (`pos-heladeria`,
incluido móvil — FR-024 exige 375px de ancho sin scroll para el enlace de login).

**Project Type**: Web application (backend FastAPI + frontend Angular, dos repositorios
independientes, siblings de este repo `pos-specs`).

**Performance Goals**: SC-002 (correo del enlace en <2 min) depende de la latencia del microservicio
de correo detrás de `EMAIL_API_URL`, fuera del control de este plan — se despacha vía Celery para no
bloquear la respuesta HTTP con ese tiempo. Sin objetivo de throughput nuevo más allá de eso.

**Constraints**:
- No se toca la forma del JWT emitido por `login()` (`RN-AUTH-06`) — el cierre de sesiones se
  resuelve comparando `tokens_valid_after` contra el claim `iat` que ya existe, no con un claim
  nuevo (research.md Decisión 1).
- El chequeo `active==True` + relectura por id de `RN-AUTH-07`/A-23 (protegida) no se modifica —
  el nuevo chequeo de `tokens_valid_after` se agrega **después**, nunca en su lugar.
- El límite de 3/15min es por correo ingresado dentro del tenant (no por IP ni dispositivo,
  Assumptions de `spec.md`) y debe aplicar igual exista o no cuenta detrás del correo (SC-003).
- `SettingsPageComponent` (config de negocio, solo `ADMIN`) no se toca ni se relaja su `roleGuard`
  — "Ajustes de cuenta" (US2) es una página nueva accesible a cajero y admin por igual (research.md
  Decisión 6).
- Fuera de alcance explícito de `spec.md`: 2FA, recuperación sin acceso al correo, preguntas de
  seguridad, historial de contraseñas, medidor de fortaleza, login sin contraseña, reset forzado
  por un admin sobre otra cuenta, caducidad periódica.

**Scale/Scope**: 1 tabla nueva (`password_reset_tokens`), 1 columna nueva (`users.tokens_valid_after`),
4 endpoints nuevos + 1 modificado en `pos-backend`; 2 pantallas nuevas (solicitar enlace, definir
contraseña nueva) + 1 pantalla nueva ("Ajustes de cuenta") + 2 modificaciones puntuales (login,
header del dashboard) en `pos-heladeria`.

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| Principio | Evaluación | Estado |
|---|---|---|
| **I. Las Nuevas Funcionalidades Nacen de un Spec** | `spec.md` existe, aprobado, con 3 historias priorizadas, 28 FRs y 4 clarificaciones ya resueltas (sesión 2026-08-24) antes de este plan. | PASS |
| **II. El Comportamiento Existente Sigue Protegido** | Tres cambios de comportamiento explícitos, ya autorizados y citados por el propio `spec.md` (encabezado + Assumptions, "Cambio de comportamiento explícito #1/#2/#3"): (1) longitud de contraseña nueva 6-128→8-12 caracteres (`RN-AUTH-01`/`RN-AUTH-09` de spec 001); (2) un cambio de contraseña exitoso ahora cierra otras sesiones (hoy no lo hace); (3) un cambio exitoso ahora dispara un correo de aviso (hoy no lo hace). Mismo criterio que specs 024/026 §Constitution Check: al ser comportamiento nuevo autorizado por un spec de evolución funcional (Principio IV) y autodocumentado en las Clarifications/Assumptions del propio `spec.md`, no exige una entrada aparte en `registro-de-anomalias.md` (ese registro es el libro de anomalías heredadas de la fase de reconocimiento, no el lugar donde se documentan decisiones de negocio de specs nuevas). `RN-AUTH-01` (verificación de `current_password`) y `RN-AUTH-02` (limpieza de `must_change_password`) se preservan sin cambios — la spec lo dice explícitamente. | PASS |
| **III. Los Characterization Tests Protegen el Comportamiento Heredado** | No existe ningún test `"CONGELA comportamiento actual:"` para `auth` (confirmado: no hay `test_auth_*.py`; spec 001 SC-004 ya señalaba esta ausencia) — no hay nada protegido que este plan pueda poner en rojo. La única regla **protegida** que este plan toca de cerca es `RN-AUTH-07`/A-23 (refresh revalida contra BD y exige `active==True`): el chequeo nuevo de `tokens_valid_after` se agrega después de ese chequeo, nunca en su lugar (research.md Decisión 1, Decisión 10), y los tests nuevos verifican explícitamente que el caso ya protegido (cuenta desactivada → `401`) sigue intacto antes de agregar el caso nuevo (revocado por cambio de contraseña). | PASS |
| **IV. Los Nuevos Specs Pueden Introducir Nuevo Comportamiento** | Toda la spec es comportamiento nuevo (Flujo A completo; endurecimiento del Flujo B) — no se exige equivalencia con el pasado, solo conformidad con `spec.md` y ausencia de regresión en `RN-AUTH-01`/`RN-AUTH-02`/`RN-AUTH-07` (preservadas explícitamente). | PASS |
| **V. Nuevas Funcionalidades Antes que Refactorizaciones Oportunistas** | No se toca `SettingsPageComponent` ni su `roleGuard` (research.md Decisión 6); no se introduce `pytest`/`TestClient` pese a que sería cómodo para probar HTTP end-to-end (research.md Decisión 10); no se generaliza `enforce()` para todo caso de rate limiting futuro, solo se agrega la función que esta spec necesita (Decisión 3); el módulo `auth` sigue sin `service.py` propio, consistente con su propia convención actual — no se le agrega una capa de servicio no pedida por ninguna FR. | PASS |
| **VI. Evolución Incremental** | El alcance se divide en las mismas unidades que las historias del spec (US1 Flujo A completo → US2 Flujo B endurecido → US3 correo de aviso, transversal a ambos), cada una con su propio archivo de test y, en el frontend, sus propias pantallas/servicios (ver Project Structure). No se mezcla con ninguna refactorización de `login`/`refresh-token` más allá del chequeo aditivo de `tokens_valid_after`. | PASS |
| **VII. Compatibilidad con Datos Históricos** | No se toca `Sale`/`Payment`/`SaleInvoice` ni ninguna venta o factura ya emitida — esta spec vive enteramente en el módulo `auth`/`users` (schema `shared`), sin relación con el ciclo de facturación. | PASS |
| **VIII. Evolución del Modelo de Datos** | Ver data-model.md: columna nueva `users.tokens_valid_after` (nullable, sin backfill, rollback `op.drop_column`) y tabla nueva `password_reset_tokens` (rollback `op.drop_table`), ambas en `shared` vía una sola migración (sin `@for_each_tenant_schema`) — estrategia de rollback explícita en research.md, sección final. | PASS |
| **IX. Dependencias Nuevas Permitidas con Justificación** | No se añade ninguna dependencia (Technical Context). Se decide explícitamente **no** adoptar `itsdangerous` pese a estar ya instalada, con alternativas consideradas (research.md Decisión 2). | PASS (no aplica) |
| **X. Verificación Obligatoria** | Cada historia de usuario tiene su "Independent Test" en `spec.md`; quickstart.md los traduce a comandos `unittest` ejecutables, más una verificación explícita de que `RN-AUTH-07`/A-23 sigue en verde tras agregar el chequeo nuevo. | PASS |
| **XI. Decisiones de Negocio Frente a Decisiones Técnicas** | Las tres decisiones de negocio (longitud 8-12, cierre de sesiones, correo de aviso) están en `spec.md`; *cómo* implementarlas sin tabla de sesiones (columna `tokens_valid_after` + relectura ya existente) y *cómo* generar el token de un solo uso (tabla con hash, no `itsdangerous`) son decisiones técnicas documentadas aparte en research.md, sin mezclarse con las de negocio. | PASS |
| **XII. Trazabilidad** | Cadena completa: `spec.md` (Necesidad+Spec+Decisión, incluida la sesión de clarificación 2026-08-24) → este `plan.md`/`research.md` (Decisión técnica) → tareas de `tasks.md` (Fase 2, no generada por este comando) → implementación → tests nuevos de `auth` + verificación explícita de `RN-AUTH-07` → `quickstart.md` (Verificación). | PASS |
| **XIII. Todo en Español de Colombia** | Este plan y todos sus artefactos (research.md, data-model.md, contracts/, quickstart.md) se escriben en español de Colombia, igual que `spec.md`. | PASS |

Sin violaciones. La tabla de Complexity Tracking al final de este documento queda vacía.

## Project Structure

### Documentation (this feature)

```text
specs/031-recuperacion-cambio-contrasena/
├── plan.md              # Este fichero (/speckit-plan)
├── research.md          # Fase 0 (/speckit-plan) — decisiones técnicas y alternativas descartadas
├── data-model.md         # Fase 1 (/speckit-plan) — entidades, columnas, transiciones, migraciones
├── quickstart.md          # Fase 1 (/speckit-plan) — validación ejecutable por historia de usuario
├── contracts/              # Fase 1 (/speckit-plan) — contratos HTTP nuevos/modificados
│   ├── forgot-password.md
│   ├── reset-password.md
│   └── change-password.md
└── tasks.md                # Fase 2 (/speckit-tasks — NO se crea en este comando)
```

### Source Code (repositorios sibling de `pos-specs`)

Esta spec vive en `pos-specs`, pero el código que describe está en `../pos-backend` y
`../pos-heladeria` (Constitución §Alcance). Rutas relativas a la raíz de cada repo.

```text
# pos-backend
app/
├── core/
│   ├── models.py               # MODIFICADO — User gana tokens_valid_after; PasswordResetToken nueva
│   ├── dependencies.py         # MODIFICADO — get_current_user/get_authenticated_user ganan el
│   │                             chequeo de tokens_valid_after (después de active==True, RN-AUTH-07)
│   ├── utils.py                 # SIN CAMBIOS — verify_password/generate_passwd_hash se reusan tal cual
│   ├── rate_limit.py            # MODIFICADO — enforce_sliding_window() nueva (ZSET), enforce()
│   │                              existente intacto
│   ├── mail.py                  # MODIFICADO — password_reset_email_body(), password_changed_email_body()
│   └── config.py                # MODIFICADO — PASSWORD_RESET_MAX_REQUESTS, _WINDOW_SECONDS,
│                                   _TOKEN_EXPIRY_MINUTES
│
├── api/v1/auth/
│   ├── routes.py                 # MODIFICADO — forgot-password, reset-password (validate+submit)
│   │                                nuevos; change_password gana FR-021/tokens_valid_after/correo;
│   │                                refresh-token gana el chequeo de tokens_valid_after
│   └── schemas.py                 # MODIFICADO — ChangePasswordRequest (8-12); ForgotPasswordRequest,
│                                     ResetPasswordRequest nuevos
│
├── characterization_tests/
│   ├── auth_fixtures.py            # NUEVO — schema_translate_map con shared+tenant, crea
│   │                                  Tenant/Role/User/PasswordResetToken en SQLite en memoria
│   ├── test_auth_forgot_password.py  # NUEVO — US1 (FR-001 a FR-012)
│   ├── test_auth_reset_password.py   # NUEVO — US1 (validar+consumir, FR-006 a FR-012, FR-019 a FR-028)
│   ├── test_auth_change_password.py  # NUEVO — US2 (FR-013 a FR-021), reemplaza la ausencia de tests
│   │                                    de spec 001 solo para lo que esta spec modifica
│   └── test_auth_session_revocation.py # NUEVO — verifica RN-AUTH-07/A-23 intacto + el caso nuevo
│                                          de tokens_valid_after en get_current_user/refresh
│
└── alembic/versions/
    └── {rev}_password_reset_tokens.py  # NUEVO — tabla + columna users.tokens_valid_after,
                                            schema shared, sin @for_each_tenant_schema

# pos-heladeria
src/app/
├── core/auth/
│   ├── auth.models.ts               # MODIFICADO — ForgotPasswordRequest, ResetPasswordRequest,
│   │                                   ValidateResetTokenResponse; ChangePasswordRequest sin cambio
│   │                                   de forma (min/max ya no se valida en el front más allá de
│   │                                   Validators.minLength(8)/maxLength(12))
│   ├── auth-api.service.ts          # MODIFICADO — forgotPassword(), validateResetToken(),
│   │                                   resetPassword()
│   └── token-storage.service.ts     # SIN CAMBIOS
│
├── core/services/
│   └── auth.service.ts               # MODIFICADO — forgotPassword()/resetPassword() nuevos;
│                                        changePassword() sigue re-logueando tras éxito (ya lo
│                                        hace hoy, research.md Decisión 1 lo reutiliza tal cual)
│
├── modules/auth/pages/
│   ├── login.component.ts             # MODIFICADO — enlace "Restablecer contraseña" bajo el
│   │                                     campo de contraseña (FR-001)
│   ├── change-password.component.ts    # MODIFICADO — Validators.minLength(8)/maxLength(12)
│   │                                     (antes 6/128), solo el rango cambia
│   ├── forgot-password.component.ts     # NUEVO — campo "Correo electrónico" + botón "Enviar
│   │                                       enlace" (FR-002), mensaje genérico (FR-003)
│   └── reset-password.component.ts       # NUEVO — valida el token al cargar (GET .../validate,
│                                            sin consumirlo), limpia cualquier sesión existente
│                                            (FR-006), formulario "Nueva contraseña"/"Confirmar
│                                            nueva contraseña" reusando el patrón de validador
│                                            cruzado de change-password.component.ts
│
├── modules/account/pages/
│   └── account-settings.component.ts      # NUEVO — "Ajustes de cuenta", sección "Cambiar
│                                             contraseña" (FR-013/FR-014), accesible a cajero y
│                                             admin (sin roleGuard de ADMIN, research.md Decisión 6)
│
├── modules/dashboard/layout/
│   └── header.component.ts                # MODIFICADO — opción "Cambiar contraseña" en el
│                                             dropdown de usuario, junto a "Cerrar sesión"
│
├── shared/password-input/
│   └── password-input.component.ts         # SIN CAMBIOS — se reusa tal cual en las 2 pantallas
│                                              nuevas (ya soporta pegar y mostrar/ocultar, FR-023/027)
│
└── app.routes.ts                            # MODIFICADO — rutas nuevas `forgot-password` (guard
                                                redirectIfAuthGuard, igual que login) y
                                                `reset-password` (pública, sin guard — limpia sesión
                                                en el propio componente en vez de redirigir);
                                                `dashboard/mi-cuenta` nueva dentro de
                                                `modules/dashboard/routes.ts`, sin roleGuard
```

**Structure Decision**: cada historia de usuario del spec se mapea a un subconjunto disjunto de los
ficheros de arriba (US1 → `forgot-password.component.ts`/`reset-password.component.ts` +
`routes.py::forgot_password/reset_password_validate/reset_password`; US2 →
`account-settings.component.ts`/`header.component.ts` + `routes.py::change_password` modificado;
US3 → `mail.py` + el despacho de correo dentro de los mismos dos flujos, sin pantalla propia). No se
crea ningún módulo/paquete nuevo en `pos-backend` más allá de un modelo (`PasswordResetToken`) y sus
fixtures/tests — se extiende el router `auth` ya existente, sin agregarle una capa `service.py` que
hoy no tiene (Principio V). En `pos-heladeria` se agrega un módulo `account` nuevo (perfil personal,
distinto de `settings`, que es configuración de negocio) y se extienden `auth`/`dashboard/layout`
con los puntos de entrada correspondientes.

## Complexity Tracking

> **Fill ONLY if Constitution Check has violations that must be justified**

Sin violaciones — tabla vacía.
