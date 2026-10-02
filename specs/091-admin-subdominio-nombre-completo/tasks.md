---

description: "Tareas de la spec 091 — acceso de Super Admin aislado en admin.skeilopos.com y nombre completo en invitaciones"
---

# Tasks: Acceso de Super Admin en `admin.skeilopos.com` y Nombre Completo en Invitaciones

**Input**: Documentos de diseño de `/specs/091-admin-subdominio-nombre-completo/`

**Prerequisites**: [plan.md](./plan.md), [spec.md](./spec.md), [research.md](./research.md) (H-1…H-12, D1–D13), [data-model.md](./data-model.md), [contracts/](./contracts/) (5 contratos), [quickstart.md](./quickstart.md)

**Tests**: La spec y el plan **sí los piden** (SC-005, Principio X; cada contrato lista sus "Pruebas requeridas"). Backend: `unittest` sobre SQLite en memoria con `app/characterization_tests/auth_fixtures.py`. Frontend: `ng test` (Vitest + jsdom). Cada prueba se escribe **antes** que su implementación y debe fallar primero. Las pruebas de reglas del nombre y de reservados recorren los **mismos vectores** en ambos repos.

**Organización**: por historia de usuario (spec.md) y agrupadas en los 3 incrementos del plan (I1 = US1+US2, I2 = US3+US4+US5, I3 = US6). Cada incremento es desplegable por separado.

## Formato: `[ID] [P?] [Story] Descripción`

- **[P]**: se puede ejecutar en paralelo (archivo distinto, sin dependencia de una tarea sin terminar)
- **[Story]**: historia de usuario (US1–US6)
- Abreviaturas de ruta: `BE/` = `../pos-backend/` · `FE/` = `../pos-heladeria/src/app/` · `SPEC/` = esta carpeta (`pos-specs/specs/091-admin-subdominio-nombre-completo/`) · `TESTS/` = `../pos-backend/app/characterization_tests/`

## Notas de gobernanza

- **Ramas (Principio XIV)**: `feat/091-platform-host-invitation-name` en `pos-backend` y en `pos-heladeria`, **desde `develop`** (ambos están hoy en `develop`). La spec vive en `pos-specs` (rama `main`).
- **Anomalías primero (Principios II/XI)**: A-99…A-102 se registran en `specs/000-reconocimiento/registro-de-anomalias.md` **antes de tocar código** (T004). Los commits que cambien un comportamiento protegido citan la anomalía.
- **Diagnóstico antes de corregir (D1, Principio X)**: el recorrido del 404 en navegador (T005) se registra en `SPEC/implementation-notes.md` **antes del primer commit de código**. Si resultara HP-2 (hosting) o HP-3 (caché) y no HP-1, **detener** la Fase 3 y revisar `research.md` D1 antes de seguir; la decisión de continuar es del usuario y T005b documenta la dependencia de despliegue que corresponda.
- **Principio III**: ningún test `CONGELA comportamiento actual:` cubre login/invitaciones/alta de negocio (H-12). Los tests existentes de invitaciones (spec 037) se **amplían**, nunca se debilitan.
- **Commits (Principio XV)** solo cuando el usuario lo pida: pequeños, en inglés, Conventional Commits, sin marcas de IA. Agrupación sugerida: (1) reservados BE; (2) login BE; (3) schema de alta BE; (4) migración+modelo; (5) validador de nombre BE; (6) invitaciones+correo BE; (7) resolver/entorno FE; (8) interceptor/guards/login FE; (9) formulario de negocio FE; (10) validador+helper FE; (11) formulario/listas FE; (12) Super Admin FE; (13) docs en `pos-specs`.
- **Fuera de alcance**: endpoints `POST/PATCH /super-admin/users` (D11, opción C), `forgot-password` del Super Admin, escape de `tenant_name`/`email`/`password` en el correo (deuda aparte), handler global de 422, migrar/borrar negocios con host reservado, validar `Origin`/`Referer`.
- **Decisión abierta O-1 (Historia 6)**: se ejecuta la opción A por defecto (solo frontend). Si el dueño de la spec elige B o C, se rehace la Fase 9 sin afectar las demás.

---

## Phase 1: Setup

**Propósito**: ramas, anomalías, diagnóstico y verificación previa de datos.

- [X] T001 Crear en `../pos-backend` la rama `feat/091-platform-host-invitation-name` **desde `develop`** (árbol limpio; hoy está en `develop`) y confirmar con `git status` que no hay cambios sin confirmar
- [X] T002 [P] Crear en `../pos-heladeria` la rama `feat/091-platform-host-invitation-name` **desde `develop`** (árbol limpio; hoy está en `develop`)
- [X] T003 [P] Crear `SPEC/implementation-notes.md` (archivo NUEVO, en español de Colombia) con las secciones "Diagnóstico del 404", "Decisiones", "Verificación automática", "Verificación manual", "Verificación en producción" y "Limitaciones"; anotar en "Decisiones" que la historia 6 sigue la opción A de research D11 por defecto (O-1 abierta)
- [X] T004 Registrar en `../pos-specs/specs/000-reconocimiento/registro-de-anomalias.md` (siguiendo el formato de las entradas A-94…A-98) las anomalías **A-99** (login de plataforma exige el host `admin`; Super Admin sin cabecera, desde host desconocido o desde el dominio raíz ya no entra), **A-100** (crear invitación exige nombre completo validado), **A-101** (reservados `admin`, `assets`, `api`, `docs` además de `www`, `app`) y **A-102** (hallazgo H-10: el formulario del Super Admin llama a `POST/PATCH /super-admin/users`, que no existen — 405 — y decisión O-1); confirmar la numeración real libre tras A-98 y ajustar las referencias en `SPEC/` si difiere; quién/cuándo: dueño de la spec, 2026-10-02
- [ ] T005 Diagnóstico del 404 **sobre el estado actual** (`develop` sin cambios), siguiendo `SPEC/quickstart.md` §2 en `https://admin.skeilopos.com/login` con DevTools (Network, "Disable cache", datos de sitio limpios): anotar en `SPEC/implementation-notes.md` si documento y `.js` responden 200, **qué petición devuelve 404 y su cuerpo** (esperado HP-1: `Tenant not found for host 'admin'`), la URL final tras el login (esperado `/dashboard/admin`) y el resultado de recargar `/super-admin/tenants`; concluir cuál hipótesis (HP-1/HP-2/HP-3) se confirma. **No usar credenciales reales en comandos ni notas**
- [X] T005b Solo si T005 confirma HP-2 (hosting) o HP-3 (caché): documentar en `SPEC/implementation-notes.md` la configuración requerida (enlace del subdominio al despliegue del SPA, relleno de SPA, purga de caché) como **dependencia de despliegue** de la spec; dejar I1 bloqueado para producción hasta que el usuario confirme que quedó aplicada; si el usuario decide seguir con la Fase 3, el código de I1 no cambia (el contexto de host del frontend, H-4, se corrige igual). Si T005 confirma HP-1, anotar "no aplica"
- [X] T006 [P] Verificación previa de datos (bloqueante para el despliegue, `SPEC/quickstart.md` §1): ejecutar contra la BD de desarrollo —y dejar anotada la consulta para ejecutarla en producción antes de desplegar— `SELECT id, name, host FROM shared.tenants WHERE lower(btrim(host)) IN ('www','app','admin','assets','api','docs');`; esperado 0 filas; registrar el resultado en `SPEC/implementation-notes.md`. Si devuelve filas, **detener** y elevar la decisión al dueño de la spec (Edge Case)

**Checkpoint**: ramas creadas, A-99…A-102 registradas, diagnóstico y verificación de datos anotados. Recién entonces se escribe código.

---

## Phase 2: Foundational (módulos puros compartidos por varias historias)

**Propósito**: los dos módulos puros del backend y los dos helpers del frontend que consumen varias historias. **Bloquea** US2 (reservados), US3/US4 (nombre) y US5/US6 (presentación).

- [X] T007 [P] Escribir `TESTS/test_reserved_hosts.py` (NUEVO): fija la lista literal `{www, app, admin, assets, api, docs}` y `PLATFORM_HOST == "admin"`; `is_reserved_subdomain` es verdadero para `admin`, `Admin`, `ADMIN`, ` admin `, `Api`, ` DOCS `, `www`, `app`, `assets` y falso para `admin2`, `admin-prueba`, `mi-api`, `docs1`, `acme` (igualdad, no prefijo) — debe fallar (módulo inexistente)
- [X] T008 Crear `BE/app/core/reserved_hosts.py` (NUEVO): `PLATFORM_HOST = "admin"`, `RESERVED_SUBDOMAINS = frozenset({"www","app","admin","assets","api","docs"})` e `is_reserved_subdomain(value: str) -> bool` (compara `value.strip().lower()`); hace pasar `test_reserved_hosts.py` (T007)
- [X] T009 [P] Escribir `TESTS/test_person_name.py` (NUEVO): recorre **todos los vectores 1–26** de `SPEC/contracts/full-name-rules.md` (válidos con su valor guardado, inválidos con el error `required`/`length`/`format` esperado), incluidos NFC (vector 9), puntos de código vs. UTF-16 (vector 26) y el orden `required → length → format` (vector 16); verifica los tres textos de mensaje literales — debe fallar (módulo inexistente)
- [X] T010 Crear `BE/app/core/person_name.py` (NUEVO): `normalize_full_name(value: str | None) -> str` (trim → NFC → obligatorio → longitud 2–100 por puntos de código → formato con rangos explícitos `A-Za-z`, `À-Ö`, `Ø-ö`, `ø-ÿ`, `Ā-ɏ`, `Ḁ-ỿ` + espacio, `'`, `’`, `-` y ≥ 2 letras) que lanza `ValueError` con los tres textos de `full-name-rules.md`; sin dependencias nuevas (`unicodedata`, `re`); hace pasar `test_person_name.py` (T009)
- [X] T011 [P] Escribir `FE/shared/validators/full-name.validator.spec.ts` (NUEVO): recorre los **mismos vectores 1–26** de `full-name-rules.md` contra `fullNameValidator` y la función pura de normalización, incluida la opción `optional` (vacío permitido; si se escribe, debe ser válido) y los tres mensajes literales — debe fallar (archivo inexistente)
- [X] T012 Crear `FE/shared/validators/full-name.validator.ts` (NUEVO): función pura `normalizeFullName(value)` que devuelve `{ value } | { error: 'required'|'length'|'format' }` con la misma regla (trim, `normalize('NFC')`, conteo por puntos de código con `Array.from`, mismos rangos de letras con expresión regular sin `\p{L}`), constante de mensajes en español de Colombia y `fullNameValidator(options?: { optional?: boolean }): ValidatorFn`; hace pasar T011
- [X] T013 [P] Escribir `FE/shared/person-display.spec.ts` (NUEVO): `displayName(name, email)` devuelve el nombre recortado, o el correo si el nombre es vacío, nulo o **igual al correo**; `initialOf` devuelve el primer **punto de código** en mayúscula (`María Pérez`→`M`, `ñandú`→`Ñ`, correo `ana@x.co`→`A`, emoji inicial sin partirlo) — debe fallar (archivo inexistente)
- [X] T014 Crear `FE/shared/person-display.ts` (NUEVO): funciones puras `displayName(name, email)` e `initialOf(name, email)` según research D9; hace pasar T013

**Checkpoint**: lista de reservados y reglas del nombre fijadas por pruebas en ambos repos; las historias pueden empezar.

---

## Phase 3: User Story 1 — El Super Admin entra por `admin.skeilopos.com/login` (Priority: P1) 🎯 MVP (Incremento I1)

**Goal**: el Super Admin abre `admin.skeilopos.com/login`, inicia sesión y llega al panel central sin 404; las rutas internas se recargan sin 404.

**Independent Test**: con `admin.localhost:4200/login` y un Super Admin → llega a `/super-admin/tenants`; recargar `/super-admin/users` no da 404 y conserva la sesión; credenciales erróneas → "credenciales inválidas"; cuenta inactiva → aviso existente (`SPEC/quickstart.md` §4 filas 1 y §5 pasos 1–2).

### Tests for User Story 1

- [X] T015 [P] [US1] Escribir `TESTS/test_auth_login_platform_isolation.py` (NUEVO), parte plataforma, reutilizando el patrón de `test_auth_login_invitation_consumption.py` (parchea `with_db` con la sesión SQLite de `auth_fixtures`): filas 1 y 2 de la tabla de `SPEC/contracts/auth-login-platform-isolation.md` (SA activa con `x-tenant-host: admin` → 200 con `is_super_admin: true`; SA inactiva → 403 `User account is inactive`), más variantes `ADMIN`, ` admin ` y `admin:4200` que también cuentan como plataforma — debe fallar donde hoy no hay reconocimiento explícito
- [X] T016 [P] [US1] Escribir `FE/core/tenant/tenant-resolver.spec.ts` (AMPLIAR el existente): casos de plataforma de la tabla de `SPEC/contracts/tenant-context-resolution.md` — `admin.skeilopos.com`, `admin.localhost`, `localhost`, `127.0.0.1` → SUPER_ADMIN; mayúsculas/espacios normalizados; los casos que hoy afirman otra cosa se **actualizan citando A-99** en el mismo commit
- [X] T017 [P] [US1] Escribir `FE/core/auth/auth-token.interceptor.spec.ts` (AMPLIAR): en contexto SUPER_ADMIN la petición lleva `X-Tenant-Host: admin`; en TENANT lleva `<slug>`; en UNRECOGNIZED no lleva cabecera; rutas del comensal y refresco sin cambio
- [X] T018 [P] [US1] Escribir `FE/modules/auth/pages/login.component.spec.ts` (AMPLIAR): en contexto plataforma se muestra la insignia "Acceso Super Admin" y, tras login de Super Admin, se navega al panel (`/super-admin/...`), no a `/dashboard/admin`

### Implementation for User Story 1

- [X] T019 [US1] Modificar `login()` en `BE/app/api/v1/auth/routes.py` (hoy `:135-157`): conservar `host = host_header.split(":", 1)[0]` **sin normalizar** para la búsqueda de negocio (`Tenant.host == host`, igualdad exacta como hoy, FR-010) y calcular aparte `is_platform = host.strip().lower() == PLATFORM_HOST` solo para reconocer la plataforma; si `is_platform` buscar solo `tenant_id IS NULL`, sin invitación; (el resto de ramas en US2); conservar respuesta 200, claims, 403 de inactiva y 503/500 de BD; hace pasar T015. **Cita A-99 en el commit**
- [X] T020 [P] [US1] Añadir `platformSlug: string` a `FE/core/tenant/app-environment.interface.ts` y `platformSlug: 'admin'` a ambos entornos `../pos-heladeria/src/environments/environment.ts` y `environment.development.ts`
- [X] T021 [US1] Modificar `FE/core/tenant/tenant-resolver.ts` y `tenant-context.model.ts`: reconocer `admin.<rootDomain>` y `admin.localhost` como SUPER_ADMIN (vía `platformSlug`), (**corregido tras la aclaración del 2026-10-02**: `localhost`/`127.0.0.1` ya NO son SUPER_ADMIN y se elimina `devRootHosts`); recibir `platformSlug` desde `tenant.initializer.ts`; hace pasar T016 (las variantes TENANT/UNRECOGNIZED se completan en US2)
- [X] T022 [US1] Modificar `FE/core/tenant/tenant-context.service.ts`: añadir `tenantHostHeader()` (devuelve `slug` en TENANT, `platformSlug` en SUPER_ADMIN, `null` en el resto) y `isTenant()`; `isSuperAdmin` y `tenantSlug` conservan su significado
- [X] T023 [US1] Modificar `FE/core/auth/auth-token.interceptor.ts` (`decorate`): usar `tenantHostHeader()` en vez de `tenantSlug()` y mandar `X-Tenant-Host` solo si no es `null`; hace pasar T017
- [X] T024 [US1] Ajustar `FE/modules/auth/pages/login.component.ts` (`:188` aprox.): la redirección tras login y la insignia se deciden por el contexto SUPER_ADMIN del resolver (que ya reconoce `admin`); hace pasar T018
- [X] T025 [US1] Verificar en `FE/core/tenant/guards/super-admin-domain.guard.ts` que sigue exigiendo `isSuperAdmin()` (sin cambio de código esperado) y que `FE/app.routes.ts` lleva a `/super-admin/tenants` en contexto plataforma; si requiere un ajuste mínimo, hacerlo aquí

**Checkpoint (I1 parcial)**: en `admin.localhost:4200` el Super Admin entra y llega al panel; rutas internas recargan (verificado en T043).

---

## Phase 4: User Story 2 — Aislamiento estricto entre plataforma y negocios (Priority: P1) (Incremento I1)

**Goal**: cuentas de plataforma solo en `admin`; cuentas de negocio solo en su subdominio; dominio raíz y hosts desconocidos sin login; `admin`, `assets`, `api`, `docs` reservados en el alta de negocio. Rechazos indistinguibles de "credenciales inválidas".

**Independent Test**: los cuatro cruces (negocio→admin, SA→negocio, SA→subdominio inexistente, alta con `admin`) se rechazan; el acceso legítimo de cada tipo sigue funcionando (`SPEC/quickstart.md` §4 y §5).

### Tests for User Story 2

- [X] T026 [P] [US2] Completar `TESTS/test_auth_login_platform_isolation.py` con las filas 3–8, 10 y 11 de `SPEC/contracts/auth-login-platform-isolation.md` (UN→`admin` 401; `acme`+SA 401; `no-existe`+SA y +UN 401; cabecera ausente/vacía 401; `acme`+UN 200 sin cambio; invitación pendiente en `acme` se consume; `admin`+correo con invitación 401), más: (a) los cuatro cuerpos de rechazo son idénticos byte a byte (I-1), (b) un rechazo no crea `User` ni consume invitación (I-3), (c) un negocio con host `admin-prueba` no se confunde con la plataforma, (d) **FR-007 / I-5**: con las dependencias reales, un token de Super Admin (`is_super_admin: true`) llamando a `get_current_user` con el host de un negocio → 401 (no existe un usuario de ese negocio con ese correo) y un token de usuario de negocio llamando a `get_current_super_admin` → 403 `Super admin access required` (estas pruebas **verifican** el comportamiento actual, H-6, y deben pasar sin cambios de código), (e) un negocio con host guardado `Acme` (mayúscula) sigue resolviéndose con `x-tenant-host: Acme` como hoy (FR-010)
- [X] T027 [P] [US2] Añadir a `TESTS/test_reserved_hosts.py` las pruebas del alta de negocio contra el schema `TenantCreateWithUser`: `admin`, `ADMIN`, ` api `, `assets`, `docs` → `ValidationError` con `loc == ("host",)` y mensaje `«<valor normalizado>» es una palabra reservada y no puede usarse como subdominio`; `admin2`, `mi-api`, `docs1`, `acme` pasan; el valor guardado no se transforma; con el endpoint (TestClient o llamada directa al schema) no se invoca `tenant_create` ante un reservado
- [X] T028 [P] [US2] Completar `FE/core/tenant/tenant-resolver.spec.ts` con el resto de la tabla de `tenant-context-resolution.md`: `acme.skeilopos.com`/`acme.localhost` → TENANT; raíz `skeilopos.com`, `www.`, `app.`, `assets.`, `api.`, `docs.` (con `.skeilopos.com` y `.localhost`) e IP/`*.pages.dev`/dominio ajeno → UNRECOGNIZED; multinivel `x.admin.skeilopos.com` → TENANT `x`; **citar A-99**
- [X] T029 [P] [US2] Crear `FE/core/tenant/guards/tenant-domain.guard.spec.ts` (NUEVO): `tenantDomainGuard` rechaza (redirige a `/login`) en UNRECOGNIZED y en SUPER_ADMIN y deja pasar solo en TENANT
- [X] T030 [P] [US2] Ampliar `FE/modules/auth/pages/login.component.spec.ts`: en UNRECOGNIZED se muestra la tarjeta "Esta dirección no corresponde a ningún acceso", **sin formulario** y sin llamadas al API
- [X] T031 [P] [US2] Crear `FE/core/tenant/reserved-slugs.parity.spec.ts` (NUEVO): `reservedSlugs` de `environment.ts` y de `environment.development.ts` son iguales a la lista literal `www, app, admin, assets, api, docs` y `platformSlug === 'admin'`
- [X] T032 [P] [US2] Crear `FE/modules/super-admin/components/tenant-form.component.spec.ts` (NUEVO): el campo Host rechaza `admin`, `Docs`, ` api ` con «…es una palabra reservada y no puede usarse como subdominio», bloquea el envío mientras sea inválido, acepta `admin2`; la sugerencia automática desde el nombre "Admin" (`toSlug`) también se marca como inválida

### Implementation for User Story 2

- [X] T033 [US2] Completar `login()` en `BE/app/api/v1/auth/routes.py`: tabla de decisión completa — host de negocio registrado (igualdad exacta con `Tenant.host`) → solo ese negocio con consumo de invitación como hoy; **host ausente, vacío, desconocido o raíz → 401 `Invalid credentials` inmediato sin consultar cuentas**; todos los rechazos con el mismo cuerpo; registrar en el log (`info`) el `reason` interno (`platform`/`tenant`/`no_host`/`unknown_host`) sin contraseña (I-2); un solo ámbito por petición (I-4); hace pasar T026. **Cita A-99**
- [X] T034 [US2] Modificar `BE/app/api/v1/super_admin/schemas.py` (`TenantCreateWithUser`, validador de `host` hoy `:191-197`): rechazar con `ValueError("«<normalizado>» es una palabra reservada y no puede usarse como subdominio")` si `is_reserved_subdomain(host)`; conservar el valor original sin transformar; hace pasar T027. **Cita A-101**
- [X] T035 [US2] Completar `FE/core/tenant/tenant-context.model.ts` (variante `{ kind: 'UNRECOGNIZED', hostname }`) y `FE/core/tenant/tenant-resolver.ts` con los tres estados de la tabla (dominio raíz, reservados distintos de `admin` y host desconocido → UNRECOGNIZED; retirar el `console.warn` de "host desconocido ⇒ SuperAdmin"); hace pasar T028
- [X] T036 [P] [US2] Actualizar `reservedSlugs` a `['www','app','admin','assets','api','docs']` en `../pos-heladeria/src/environments/environment.ts` y `environment.development.ts`; hace pasar T031
- [X] T037 [US2] Modificar `FE/core/tenant/guards/tenant-domain.guard.ts`: pasar **solo** si `isTenant()`; en otro caso redirigir a `/login`; hace pasar T029
- [X] T038 [US2] Modificar `FE/modules/auth/pages/login.component.ts`: en UNRECOGNIZED mostrar la tarjeta "Esta dirección no corresponde a ningún acceso" sin formulario y sin llamar al API; hace pasar T030
- [X] T039 [US2] Modificar `FE/modules/super-admin/components/tenant-form.component.ts`: validador del campo Host contra `environment.reservedSlugs` (mismo recorte y minúsculas), mensaje «<valor> es una palabra reservada y no puede usarse como subdominio», bloqueo del envío y lectura del 422 con arreglo del servidor (`detail[0].msg` sin prefijo `Value error, `); hace pasar T032. **Cita A-101**
- [X] T040 [US2] Confirmar en `FE/modules/menu/` (rutas `menu/t/:token` del comensal) con una prueba existente o una prueba nueva mínima que las rutas públicas siguen funcionando en un host UNRECOGNIZED (no usan `X-Tenant-Host`); si ninguna prueba existente lo cubre, añadirla junto a `tenant-resolver.spec.ts`

**Checkpoint (I1 completo)**: aislamiento verificado por pruebas en ambos repos. Incremento desplegable solo.

---

## Phase 5: Verificación del Incremento I1

**Propósito**: cierre verificable de US1+US2 antes de pasar a I2 (Principio X).

- [X] T041 Ejecutar `python -m unittest app.characterization_tests.test_auth_login_platform_isolation app.characterization_tests.test_reserved_hosts -v` en `../pos-backend` y registrar el resultado en `SPEC/implementation-notes.md`
- [X] T042 Ejecutar la suite completa `python -m unittest discover -s app/characterization_tests` (o el comando habitual del repo) en `../pos-backend` y `ng test` + `ng build` en `../pos-heladeria`; ningún test `CONGELA` en rojo; registrar conteos en `SPEC/implementation-notes.md`
- [X] T043 Aislamiento por API con `curl` (`SPEC/quickstart.md` §4: seis peticiones de login y reservados `admin`, `ADMIN`, ` api `, `assets`, `docs` → 422, `admin2`/`mi-api` pasan) y recorrido manual en navegador `SPEC/quickstart.md` §5 pasos 1–9 sobre `admin.localhost:4200`, `localhost:4200`, `acme.localhost:4200`, `www.localhost:4200`; comprobar que los cuatro 401 tienen cuerpo idéntico; anotar resultados (con DevTools, no solo `curl`) en `SPEC/implementation-notes.md` §"Verificación manual". **No ejecutar contra producción con credenciales reales**

**Checkpoint**: I1 listo para desplegar (orden de despliegue en T079).

---

## Phase 6: User Story 3 — Invitar registrando el nombre completo (Priority: P1) (Incremento I2)

**Goal**: el formulario "Invitar usuario" captura "Nombre completo"; el correo saluda por nombre; la invitación lo conserva (también al reenviar); la cuenta nace con ese nombre al aceptar.

**Independent Test**: invitar a "María Pérez" → el correo dice "Hola, María Pérez:"; al primer login de la persona la lista de usuarios muestra "María Pérez" con avatar "M"; `José Ñañez O'Brien-Díaz` se guarda y se muestra idéntico (`SPEC/quickstart.md` §6 filas 1 y 4).

### Tests for User Story 3

- [X] T044 [P] [US3] Escribir `TESTS/test_invitation_email_greeting.py` (NUEVO): `invitation_email_body(..., name="María Pérez")` contiene `Hola, María Pérez:`; con `name="O'Brien"` contiene `O&#x27;Brien` y no el apóstrofe crudo; con `name=None` contiene `Hola:` y no un saludo con el correo; el resto del cuerpo (tabla de datos de acceso, aviso de contraseña temporal) no cambia; asunto sin cambio; los llamadores sin `name` siguen funcionando
- [X] T045 [P] [US3] Ampliar `TESTS/test_invitations_create.py`: todos los casos de creación envían `name`; una invitación con `name` válido persiste `UserInvitation.name` recortado y en NFC, el correo se arma con ese nombre, la respuesta 201 incluye `name`; dos invitaciones con el mismo nombre y correos distintos se aceptan (Historia 4.8); `José Ñañez O'Brien-Díaz` se guarda idéntico
- [X] T046 [P] [US3] Ampliar `TESTS/test_invitations_resend_cancel.py`: reenviar una invitación con nombre conserva el nombre y el correo reenviado saluda por él; reenviar una **anterior sin nombre** (`name` nulo) envía `Hola:`; el nombre no cambia al reenviar
- [X] T047 [P] [US3] Ampliar `TESTS/test_auth_login_invitation_consumption.py`: al consumirse una invitación con nombre, `User.name == invitation.name`; con invitación anterior (`name` nulo) `User.name == invitation.email` como hoy (FR-024, SC-007); ningún test existente se debilita
- [X] T048 [P] [US3] Ampliar `TESTS/test_invitations_list.py`: cada elemento incluye `name` (cadena o `null`), sin cambio de paginación ni de parámetros
- [X] T049 [P] [US3] Escribir pruebas en `FE/modules/users/` (NUEVO `components/invitation-form.component.spec.ts`): el campo "Nombre completo" aparece **primero**, antes de correo y rol; con nombre válido el envío manda `name` ya recortado en el payload; el servicio `InvitationsService` expone `name` en `PendingInvitation` e `InvitationCreatePayload`

### Implementation for User Story 3

- [X] T050 [US3] Añadir `name: Mapped[Optional[str]] = mapped_column(String(100), nullable=True)` a `UserInvitation` en `BE/app/core/models.py` (junto a `:205-260`) y crear la migración `BE/alembic/versions/<rev>_091_invitation_full_name.py` con `down_revision = 'a89f0c1d2e3b'`, `upgrade()` = `op.add_column('user_invitations', sa.Column('name', sa.String(100), nullable=True), schema='shared')` y `downgrade()` = `op.drop_column('user_invitations', 'name', schema='shared')`; sin backfill ni índice (`SPEC/data-model.md`); verificar `alembic heads` con cabecera única, `upgrade head` y `downgrade -1` en la BD de desarrollo
- [X] T051 [US3] Modificar `invitation_email_body` en `BE/app/core/mail.py` (hoy `:66-88`): nueva firma `invitation_email_body(tenant_name, login_url, email, password, name=None)` que antepone `Hola, <html.escape(name, quote=True)>:` o `Hola:` si no hay nombre; el resto sin cambio y **sin** escapar `tenant_name`/`email`/`password` (Principio V); hace pasar T044
- [X] T052 [US3] Modificar `BE/app/api/v1/invitations/schemas.py`: `InvitationCreate.name` declarado con `Field(default=None, validate_default=True)` y un `field_validator("name", mode="before")` que llama a `normalize_full_name` (omitir la clave, vacío y solo espacios dan el mismo "El nombre es obligatorio"); `InvitationResponse.name: Optional[str]`; descripción OpenAPI "obligatorio"
- [X] T053 [US3] Modificar `BE/app/api/v1/invitations/router.py`: persistir `name` al crear (la validación del cuerpo corre **antes** de límite del plan/correo repetido/rol, así un nombre inválido no crea invitación, no envía correo y no consume cupo), pasar `name` al correo en alta y reenvío, devolver `name` en alta, lista y reenvío; hace pasar T045, T046, T048. **Cita A-100**
- [X] T054 [US3] Modificar `BE/app/api/v1/auth/routes.py` (consumo de invitación, hoy `:117`): `User(name=invitation.name or invitation.email)`; hace pasar T047
- [X] T055 [US3] Modificar `FE/modules/users/services/invitations.service.ts` y las interfaces asociadas: `PendingInvitation.name: string | null`, `InvitationCreatePayload.name`; `extractError` lee `detail[0].msg` cuando `detail` es un arreglo y retira el prefijo `Value error, `
- [X] T056 [US3] Modificar `FE/modules/users/components/invitation-form.component.ts`: añadir el campo "Nombre completo" **primero** (antes de correo y rol) con `fullNameValidator()`, mostrar el texto del primer error al tocar o enviar, bloquear el envío mientras sea inválido y enviar `name` ya recortado/normalizado; hace pasar T049 (la limpieza FR-023 se completa en US5)

**Checkpoint**: una invitación con nombre recorre alta → correo → aceptación → cuenta con ese nombre.

---

## Phase 7: User Story 4 — Validación del nombre (Priority: P1) (Incremento I2)

**Goal**: el formulario avisa de inmediato y el servidor rechaza (422) todo nombre inválido aunque se salten la pantalla; nada se guarda ni se envía.

**Independent Test**: vacío, solo espacios, 1 carácter, 101 caracteres, `<script>alert(1)</script>`, `Ana3`, `Ana@` y una petición directa sin `name` → todos bloquean con el mensaje correcto y sin correo (`SPEC/quickstart.md` §6 filas 2, 3, 5, 6).

### Tests for User Story 4

- [X] T057 [P] [US4] Ampliar `TESTS/test_invitations_create.py` con la tabla de 422: ausente, `""`, `"   "` → "El nombre es obligatorio"; `"A"` y 101 letras → "El nombre debe tener entre 2 y 100 caracteres"; `<script>alert(1)</script>`, `Ana3`, `Ana@`, `Иван`, `Ana 😀` → mensaje de formato; en **cada** caso: `detail[0].loc == ["body","name"]`, `msg` empieza con `Value error, `, **no** se crea invitación, **no** se llama al envío de correo y **no** se consume cupo del plan; `" Ana "` se guarda como `Ana`
- [X] T058 [P] [US4] Ampliar `FE/modules/users/components/invitation-form.component.spec.ts`: vacío/solo espacios → "El nombre es obligatorio"; `A` y 101 → mensaje de longitud; `<script>…`, `Ana3`, `Ana@` → mensaje de formato; envío bloqueado en todos; un 422 del servidor con `detail` arreglo se muestra sin el prefijo `Value error, `

### Implementation for User Story 4

- [X] T059 [US4] Verificar que `BE/app/api/v1/invitations/router.py` y `schemas.py` (T052/T053) cumplen T057 y, si algún caso no lo hace (p. ej. orden de comprobaciones), corregirlo ahí; ejecutar `python -m unittest app.characterization_tests.test_invitations_create app.characterization_tests.test_invitations_list app.characterization_tests.test_invitations_resend_cancel app.characterization_tests.test_auth_login_invitation_consumption app.characterization_tests.test_invitation_email_greeting app.characterization_tests.test_person_name -v`
- [X] T060 [US4] Verificar que `FE/modules/users/components/invitation-form.component.ts` (T056) cumple T058; ajustar mensajes/estados de error si algún caso no pasa

**Checkpoint**: SC-005 cumplido en pantalla y en petición directa.

---

## Phase 8: User Story 5 — Lista de usuarios con nombre, avatar y compatibilidad (Priority: P2) (Incremento I2)

**Goal**: la lista de usuarios y la de invitaciones pendientes muestran nombre e inicial; las cuentas anteriores se ven con su correo; el formulario queda limpio al cancelar.

**Independent Test**: "Usuarios" con cuentas antiguas (nombre = correo) y nuevas: antiguas muestran correo e inicial del correo, nuevas nombre e inicial; Cancelar y reabrir deja todo vacío (`SPEC/quickstart.md` §6 filas 7, 8, 10, 11).

### Tests for User Story 5

- [X] T061 [P] [US5] Ampliar `FE/modules/users/pages/users-page.component.spec.ts`: usuario con nombre `María Pérez` → título `María Pérez` y avatar `M`; usuario con nombre igual al correo o vacío → título = correo y avatar = inicial del correo; ninguna excepción; un nombre `O'Brien-Díaz` se muestra idéntico y uno con `<b>` (si llegara guardado) se renderiza como texto, no como HTML (FR-017, SC-008); nombre de 100 caracteres con las clases `truncate`/`min-w-0`
- [X] T062 [P] [US5] Crear `FE/modules/users/components/pending-invitations-list.component.spec.ts` (NUEVO): con `name` muestra el nombre como título y el correo debajo; con `name === null` muestra solo el correo; `O'Brien-Díaz` se muestra idéntico; nombre largo no desborda
- [X] T063 [P] [US5] Ampliar `FE/modules/users/components/invitation-form.component.spec.ts` (FR-023, Escenario 9): escribir nombre + correo + rol y un error de servidor → **Cancelar** → reabrir: todos los campos vacíos y `invitationsService.error()` es `null`; lo mismo tras un envío exitoso y al iniciar el componente

### Implementation for User Story 5

- [X] T064 [US5] Modificar `FE/modules/users/pages/users-page.component.ts` (hoy `:108-112` pinta `user.name` y `user.name.charAt(0)`): título = `displayName(user.name, user.email)`, avatar = `initialOf(user.name, user.email)` en mayúscula; contenedor con `min-w-0` y título con `truncate`; hace pasar T061
- [X] T065 [US5] Modificar `FE/modules/users/components/pending-invitations-list.component.ts`: nombre (si hay) como título y correo debajo con `truncate`/`min-w-0`; solo el correo si `name` es `null`; hace pasar T062
- [X] T066 [US5] Modificar `FE/modules/users/components/invitation-form.component.ts` y `FE/modules/users/pages/users-page.component.ts`: `form.reset()` e `invitationsService.error.set(null)` al **Cancelar**, tras **envío exitoso** y al **abrir** el formulario (D10); hace pasar T063
- [X] T067 [US5] Revisar `FE/modules/users/` por otros lugares que pinten el nombre del usuario o de la invitación con `charAt(0)` / `.name` sin helper (búsqueda con `grep`) y sustituir por `displayName`/`initialOf`; anotar los hallazgos en `SPEC/implementation-notes.md`

### Verificación del Incremento I2

- [X] T068 Ejecutar la suite completa de `characterization_tests` en `../pos-backend` y `ng test` + `ng build` en `../pos-heladeria`; registrar conteos en `SPEC/implementation-notes.md`; confirmar que ningún test existente se debilitó (`git diff` de `TESTS/` solo añade casos o el campo `name`)
- [ ] T069 Recorrido manual `SPEC/quickstart.md` §6 filas 1–12 como ADMIN de un negocio de prueba (correo en bandeja de pruebas): el saludo "Hola, María Pérez:", los 13 escenarios del brief, reenvío con y sin nombre, aceptación de una invitación **anterior** sin nombre; anotar resultados en `SPEC/implementation-notes.md` §"Verificación manual"

**Checkpoint (I2 completo)**: nombre completo de punta a punta; la migración (T050) se despliega antes que el backend.

---

## Phase 9: User Story 6 — Nombre completo en el dashboard del Super Admin (Priority: P2) (Incremento I3)

**Goal**: el formulario de usuarios del Super Admin aplica el mismo validador (nombre obligatorio al **crear**; al **editar**, opcional pero válido si se escribe) y su tabla usa el helper de presentación. **Solo frontend** (opción A de D11; el guardado seguirá fallando con 405 hasta que exista el endpoint, A-102).

**Independent Test**: en `admin.localhost:4200/super-admin/users`, "Nuevo usuario" con nombre vacío/inválido muestra los mismos mensajes; "Editar" de un usuario sin nombre guarda sin bloquearse por el nombre; la tabla muestra nombre e inicial de los usuarios existentes (`SPEC/quickstart.md` §7).

### Tests for User Story 6

- [X] T070 [P] [US6] Crear `FE/modules/super-admin/components/admin-user-form.component.spec.ts` (NUEVO): en modo **crear**, nombre vacío/solo espacios → "El nombre es obligatorio", 1 y 101 caracteres → mensaje de longitud, `Ana3` → mensaje de formato, envío bloqueado; en modo **editar** un usuario sin nombre propio, guardar sin tocar el nombre **no** se bloquea, y si se escribe uno inválido sí; nombre válido se envía recortado
- [X] T071 [P] [US6] Crear `FE/modules/super-admin/pages/super-admin-users-page.component.spec.ts` (NUEVO): la tabla muestra `displayName`/`initialOf` (nombre e inicial; correo e inicial del correo si no hay nombre propio), sin errores con nombres vacíos o iguales al correo

### Implementation for User Story 6

- [X] T072 [US6] Modificar `FE/modules/super-admin/components/admin-user-form.component.ts`: aplicar `fullNameValidator()` en creación y `fullNameValidator({ optional: true })` en edición, con los mismos mensajes; hace pasar T070
- [X] T073 [US6] Modificar `FE/modules/super-admin/pages/super-admin-users-page.component.ts`: título = `displayName`, avatar = `initialOf`, con `truncate`/`min-w-0`; hace pasar T071
- [ ] T074 [US6] Recorrido manual `SPEC/quickstart.md` §7 y registrar en `SPEC/implementation-notes.md` el resultado, el aviso de que el guardado responde 405 (hallazgo H-10, A-102) y la **decisión O-1 pendiente del dueño de la spec** (A, B o C); si el dueño elige B o C, abrir spec propia y marcar esta fase como superada

**Checkpoint (I3)**: validación y presentación aplicadas en el Super Admin; guardado dependiente de O-1.

---

## Phase 10: Polish y despliegue

**Propósito**: cierre transversal y comprobación en producción.

- [X] T075 [P] Actualizar `SPEC/implementation-notes.md` con la sección "Decisiones" final (diagnóstico real del 404, hipótesis confirmada, hallazgos nuevos), "Limitaciones" (`X-Tenant-Host` es política y no frontera criptográfica; ventana de 422 con SPA antiguo en caché; O-1) y las desviaciones respecto del plan, si las hubo
- [X] T076 [P] Añadir a `../pos-specs/specs/000-reconocimiento/registro-de-anomalias.md` el estado final de A-99…A-102 (resuelta / abierta O-1) con la referencia a las tareas y commits
- [X] T077 Revisión de paridad: confirmar que la lista literal de reservados (T007 / T031) y los vectores del nombre (T009 / T011) son idénticos en ambos repos y coinciden con `SPEC/contracts/`; ejecutar `grep -rn "'admin'\|\"admin\"" ` en `BE/app` y `FE/` para detectar usos sueltos de la cadena que debieran usar `PLATFORM_HOST`/`platformSlug`
- [X] T078 Barrido de seguridad (Principio X): confirmar que ningún log nuevo escribe contraseñas ni tokens, que el 401 por aislamiento no revela el motivo y que `html.escape` se usa en el saludo; confirmar que no se añadieron dependencias (`git diff` de `requirements*.txt` y `package.json` vacío)
- [X] T079 Preparar el plan de despliegue en `SPEC/implementation-notes.md` con el orden de `research.md` D12: **(1)** consulta del T006 en producción (0 filas) → **(2)** migración (T050) → **(3)** backend → **(4)** frontend, **3 y 4 seguidos**; incluir rollback (revertir front y back, luego `alembic downgrade -1`)
- [ ] T080 Tras desplegar (solo cuando el usuario lo indique; **sin credenciales reales en comandos**): ejecutar `SPEC/quickstart.md` §8 — `curl` a `https://admin.skeilopos.com/login`, `/super-admin/tenants`, `/super-admin/users`, `/super-admin/plans` (ajustar a las rutas reales de `FE/app.routes.ts`) y a una ruta inexistente → 200 con el HTML del SPA (SC-001); login con `X-Tenant-Host: zzz-no-existe` y sin cabecera → 401 `Invalid credentials` — y en navegador real el recorrido 1–2 de §5 en `https://admin.skeilopos.com` más un login normal en un negocio existente (SC-003); anotar en `SPEC/implementation-notes.md` §"Verificación en producción"
- [ ] T081 Criterios de salida (`SPEC/quickstart.md` §9): bloque 1 con 0 filas; bloque 2 registrado; bloque 3 en verde; bloques 4–6 con el resultado esperado (13 escenarios cubiertos); bloque 7 con O-1 anotada; bloque 8 OK; marcar la spec lista para revisión del usuario (sin commit ni push salvo petición expresa)

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Fase 1)**: sin dependencias. T004 (anomalías) y T005 (diagnóstico) preceden a **todo** el código; T006 precede al despliegue
- **Foundational (Fase 2)**: depende de T001/T002 (ramas) y T004. **Bloquea**: US2 (T008, T036), US3/US4 (T010, T012), US5/US6 (T014)
- **US1 (Fase 3) y US2 (Fase 4)** — Incremento I1: US1 y US2 comparten `login()` y el resolver, así que **US2 se hace después de US1** (T033 completa T019; T035 completa T021)
- **Verificación I1 (Fase 5)**: depende de US1+US2
- **US3 (Fase 6) → US4 (Fase 7) → US5 (Fase 8)** — Incremento I2: US3 introduce el campo, el schema y el formulario; US4 verifica y cierra las reglas sobre esa misma implementación; US5 depende del formulario (T056) y de `person-display` (T014)
- **I2 no depende de I1** a nivel funcional (se pueden hacer en paralelo con distinto desarrollador), salvo que ambos tocan `BE/app/api/v1/auth/routes.py` (T019/T033 y T054) → **secuenciar esos tres cambios** o coordinar el merge
- **US6 (Fase 9)** — Incremento I3: depende de T012 y T014 (helpers) y es independiente del resto; sujeta a la decisión O-1
- **Polish (Fase 10)**: depende de las historias deseadas; T080 solo tras desplegar

### User Story Dependencies

- **US1 (P1)**: tras Foundational (no usa T008/T010)
- **US2 (P1)**: tras US1 (comparte `login()` y resolver) y T008
- **US3 (P1)**: tras T010, T012; independiente de I1
- **US4 (P1)**: tras US3 (verifica la misma implementación)
- **US5 (P2)**: tras US3 y T014
- **US6 (P2)**: tras T012 y T014; independiente de US3–US5

### Within Each User Story

- Prueba primero (debe fallar) → implementación → prueba en verde → verificación manual del incremento
- Modelo/migración (T050) antes de schemas/router (T052–T054); backend antes de frontend cuando el frontend consume el campo nuevo

### Parallel Opportunities

- Setup: T002, T003 y T006 en paralelo tras T001
- Foundational: los cuatro pares prueba+módulo son independientes entre sí (T007–T008, T009–T010, T011–T012, T013–T014); dentro de cada par, la prueba va primero
- US1: T015–T018 (pruebas) en paralelo; T020 en paralelo con T019
- US2: T026–T032 (pruebas) en paralelo; T036 en paralelo con T034/T035
- US3: T044–T049 (pruebas) en paralelo
- US5: T061–T063 en paralelo
- US6: T070 y T071 en paralelo
- Polish: T075 y T076 en paralelo

---

## Parallel Example: User Story 2

```bash
# Pruebas de US2 en paralelo (archivos distintos):
Task: "T026 Completar TESTS/test_auth_login_platform_isolation.py (filas 3–8, 10, 11)"
Task: "T027 Añadir pruebas del alta de negocio a TESTS/test_reserved_hosts.py"
Task: "T028 Completar FE/core/tenant/tenant-resolver.spec.ts (tabla completa)"
Task: "T029 Crear FE/core/tenant/guards/tenant-domain.guard.spec.ts"
Task: "T031 Crear FE/core/tenant/reserved-slugs.parity.spec.ts"
Task: "T032 Crear FE/modules/super-admin/components/tenant-form.component.spec.ts"
```

---

## Implementation Strategy

### MVP First (US1 + US2 = Incremento I1)

1. Fase 1 (ramas, **A-99…A-102**, **diagnóstico del 404**, verificación de datos)
2. Fase 2 (reservados + helpers; el MVP solo necesita T007–T008)
3. Fase 3 (US1) y Fase 4 (US2)
4. **DETENER y VALIDAR** con la Fase 5 (aislamiento por API y navegador)
5. Desplegar I1 solo si el usuario lo pide (no depende de la migración)

### Incremental Delivery

1. I1 (US1+US2) → verificar → desplegable
2. I2 (US3+US4+US5) → migración + backend + frontend → verificar → desplegable
3. I3 (US6) → solo frontend; sujeto a O-1
4. Cada incremento aporta valor sin romper el anterior; el orden de despliegue de I2 es migración → backend → frontend (research D12)

### Parallel Team Strategy

1. Equipo completa Setup + Foundational
2. Después: dev A → I1 (US1 → US2); dev B → I2 (US3 → US4 → US5); dev C → I3 (US6)
3. Coordinar el merge de `BE/app/api/v1/auth/routes.py` (T019, T033, T054)

---

## Notes

- [P] = archivos distintos, sin dependencia de una tarea sin terminar
- La etiqueta [Story] traza cada tarea a su historia de `spec.md`
- Verificar que cada prueba falla antes de implementar; ninguna prueba se debilita (Principio III)
- Commit solo cuando el usuario lo pida (Principio XV): pequeño, en inglés, Conventional Commits, sin marcas de IA
- Todo texto de pantalla, mensajes de error, correo, comentarios y documentación en español de Colombia (Principio XIII)
- Evitar: tareas vagas, conflictos de archivo entre tareas [P], ejecutar contra producción con credenciales reales, cambios fuera de alcance (endpoints `POST/PATCH /super-admin/users`, handler global de 422)
