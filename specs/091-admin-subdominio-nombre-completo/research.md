# Research: Acceso de Super Admin en `admin.skeilopos.com` y Nombre Completo en Invitaciones

**Spec**: [spec.md](./spec.md) | **Plan**: [plan.md](./plan.md) | **Fecha**: 2026-10-02

Todo lo de abajo sale de leer `pos-backend` y `pos-heladeria` (ambos en `develop`, árbol limpio) y de **sondas HTTP de solo lectura** a producción hechas el 2026-10-02. No quedó ningún `NEEDS CLARIFICATION`; sí queda **una decisión abierta de alcance** (O-1, Historia 6), con recomendación.

## Hechos verificados (base de las decisiones)

| # | Hecho | Evidencia |
|---|---|---|
| H-1 | El **login** decide el ámbito por la cabecera `x-tenant-host`: si el host no es un negocio registrado **o falta la cabecera**, busca la cuenta entre los usuarios con `tenant_id IS NULL` (Super Admin). Es la causa del hallazgo de la spec (Super Admin entra desde cualquier host inexistente o sin cabecera). | `pos-backend/app/api/v1/auth/routes.py:135-157` |
| H-2 | El **frontend** solo manda `X-Tenant-Host` cuando el contexto es negocio (`tenantSlug()`); en contexto Super Admin no manda nada. | `core/auth/auth-token.interceptor.ts` (`decorate`) |
| H-3 | El resolver del frontend trata como **SUPER_ADMIN**: el dominio raíz, `localhost`/`127.0.0.1`, los slugs reservados (`www`, `app`) y **cualquier host desconocido** (con `console.warn`). `admin` **no** está reservado: `admin.skeilopos.com` se interpreta como el negocio "admin". | `core/tenant/tenant-resolver.ts`, `environment*.ts` |
| H-4 | Consecuencia de H-2+H-3 en `admin.skeilopos.com`: el SPA manda `X-Tenant-Host: admin`; el backend no halla ese negocio y, por H-1, deja entrar al Super Admin; pero el SPA se cree en contexto **negocio** → tras el login navega a `/dashboard/admin` (guard de negocio), y cada llamada de negocio pasa por `get_tenant`, que responde **404 "Tenant not found for host 'admin'"**. | `login.component.ts:188`, `app.routes.ts`, `core/db.py:190-203` |
| H-5 | En producción, hoy `admin.skeilopos.com/login`, `/super-admin/tenants` y una ruta inexistente responden **200** con el HTML del SPA (relleno de SPA activo en el hosting). `app.`/`api.`/`assets.skeilopos.com/login` → 404; `docs.` y un subdominio inexistente → 525; `skeilopos.com/login` → **308 a `www.skeilopos.com/login` → 404** (la landing). | `curl` 2026-10-02 |
| H-6 | **FR-007 ya se cumple en el backend**: las rutas de negocio exigen `User.tenant_id == tenant.id` del host (`get_current_user`) y las de plataforma exigen `is_super_admin` y `tenant_id IS NULL` (`get_current_super_admin`). El token de un ámbito no sirve en el otro. Solo falta cerrar el **login** y el **contexto del SPA**. | `core/dependencies.py:243-333` |
| H-7 | El alta de negocio acepta cualquier `host` de ≥ 3 caracteres: sin lista de reservados ni en el schema ni en `tenant_create`. Único llamador de `tenant_create`: `POST /super-admin/tenants`. | `super_admin/schemas.py:191-197`, `core/db.py:56`, grep |
| H-8 | `UserInvitation` no tiene `name`; al consumirse crea `User(name=invitation.email)`. `User.name` es `VARCHAR(150) NOT NULL`. La lista de usuarios del negocio ya pinta `user.name` y `user.name.charAt(0)`. | `core/models.py:131,205-260`, `auth/routes.py:117`, `users-page.component.ts:108-112` |
| H-9 | `invitation_email_body(...)` no tiene saludo ni escapa nada (interpola `tenant_name`, `email`, `password` en HTML crudo). | `core/mail.py:66-88` |
| H-10 | **Hallazgo nuevo (contradice un supuesto de la spec)**: el backend **no tiene** `POST /super-admin/users` ni `PATCH /super-admin/users/{id}`; solo `GET /super-admin/users`. La spec 037 retiró la creación directa de usuarios. Sin embargo el formulario del Super Admin (`AdminUserFormComponent`, botón "Nuevo usuario" visible) los llama. Verificado: `POST https://api.skeilopos.com/api/v1/super-admin/users` → **405**. Crear, editar y activar/desactivar desde esa pantalla hoy fallan. | `super_admin/router.py` (4 rutas), `super-admin-users.service.ts`, `curl` |
| H-11 | Cabecera de la migración más reciente (`head` único): `a89f0c1d2e3b` (spec 089). | `pos-backend/alembic/versions/` |
| H-12 | Ningún test `"CONGELA comportamiento actual:"` toca `login()`, las invitaciones ni el alta de negocio (la única aparición del texto es una nota en el docstring de `test_auth_login_invitation_consumption.py`). | grep en `characterization_tests/` |

## Decisiones

### D1 — Diagnóstico del 404 (la spec exige diagnosticar antes de corregir)

**Decision**: tratar el 404 como **un síntoma con tres hipótesis**, y hacer el diagnóstico real en navegador como **primera tarea** (documentado en `implementation-notes.md`), antes de escribir código:

- **HP-1 (la más probable, con evidencia de lectura de código — H-4)**: el 404 es el del **API** (`Tenant not found for host 'admin'`) que aparece tras iniciar sesión, porque el SPA cree que `admin` es un negocio. La corrección es la del contexto (D3/D4), no del hosting.
- **HP-2 (hosting)**: en el momento del reporte el subdominio no estaba enlazado al despliegue o faltaba el relleno de SPA. **Hoy no se reproduce** (H-5: 200 en `/login` y en rutas profundas). Se confirma en el navegador y se registra; si reapareciera, es trabajo de despliegue (la spec lo declara fuera de su alcance funcional).
- **HP-3 (caché)**: service worker o caché de Cloudflare sirviendo una versión previa. Se descarta limpiando datos de sitio y revisando `age`/versión del bundle.

**Rationale**: sonda `curl` solo prueba que el HTML estático responde 200; **no** ejecuta el SPA ni el login. Declarar el problema resuelto con eso sería repetir el error de la spec 090 (verificación parcial). **Alternatives**: asumir "es el hosting" y no tocar código → deja intacto H-4 (el Super Admin seguiría rebotando a un contexto de negocio).

### D2 — Quién decide que una petición es "de plataforma": el host explícito, nunca la ausencia de host

**Decision**: el backend reconoce el ámbito de plataforma **solo** cuando `x-tenant-host` es exactamente `admin` (comparación sin distinguir mayúsculas). El frontend, en contexto Super Admin, manda `X-Tenant-Host: admin`. Reglas del login (detalle en [contracts/auth-login-platform-isolation.md](./contracts/auth-login-platform-isolation.md)):

| `x-tenant-host` | Se busca la cuenta en… | Invitación pendiente |
|---|---|---|
| `admin` | cuentas de plataforma (`tenant_id IS NULL`) | nunca |
| slug de un negocio registrado | solo ese negocio | como hoy (US037) |
| ausente, vacío, desconocido, raíz | **no se busca**: 401 inmediato | nunca |

Todo rechazo es el **mismo** `401 {"detail":"Invalid credentials"}` (FR-003/004/005/006), sin pista de si la cuenta existe en el otro ámbito. El motivo real solo va al log operativo (`reason=no_host|unknown_host|…`, sin contraseña).

**Rationale**: reutiliza el canal que ya existe (la cabecera), cambia una sola función y deja FR-007 intacto (H-6). Cumple FR-006 por construcción: el dominio raíz nunca llega con `admin`. **Limitación declarada**: la cabecera la fija el cliente, así que esto es **política de aislamiento del producto**, no una frontera criptográfica; la protección real sigue siendo credenciales + rol + `tenant_id` del token (mismo modelo de confianza que ya usa todo el API). **Alternatives**: (a) validar `Origin`/`Referer` además → rompería Swagger, curl y el desarrollo local, y no añade garantía contra un cliente malicioso; se anota como posible endurecimiento aparte. (b) Endpoint nuevo `/auth/platform-login` → duplica el flujo, cambia el contrato y el frontend; innecesario. (c) Mantener "sin cabecera = Super Admin" solo en desarrollo → bifurca el comportamiento por entorno; el modo local equivalente se resuelve en el frontend (D4).

### D3 — Lista única de reservados, en el backend y reflejada en el frontend

**Decision**: constante única `RESERVED_SUBDOMAINS = {www, app, admin, assets, api, docs}` y `PLATFORM_HOST = "admin"` en un módulo nuevo del backend (`app/core/reserved_hosts.py`). El schema `TenantCreateWithUser` rechaza el `host` reservado (compara tras `strip().lower()`), respondiendo 422 con "«admin» es una palabra reservada y no puede usarse como subdominio". En el frontend, `environment.reservedSlugs` lleva la **misma** lista (FR-009) y el formulario de alta de negocio la valida antes de enviar. Una prueba en cada repositorio fija la lista (si divergen, falla).

**Rationale**: `tenant_create` tiene un único llamador (H-7), así que el validador del schema cubre toda petición HTTP, incluida la directa que se salta la pantalla; no hace falta una segunda guarda en `core/db.py` (no se añade código sin llamador). **Alternatives**: lista en variable de entorno/BD → superficie de configuración sin necesidad (los nombres son una decisión de producto estable); servir la lista desde un endpoint al frontend → una llamada más para una constante.

### D4 — Contexto de host del frontend: tercer estado "no reconocido"

**Decision**: el resolver pasa de 2 a 3 resultados (tabla completa en [contracts/tenant-context-resolution.md](./contracts/tenant-context-resolution.md)):

- `admin.<rootDomain>` y `admin.localhost` → **SUPER_ADMIN** (plataforma).
- `localhost` y `127.0.0.1` → **UNRECOGNIZED**, como cualquier host desconocido. *(Se había propuesto un "modo local" que los trataba como SUPER_ADMIN; el usuario lo descartó el 2026-10-02: al login de plataforma solo se llega con la URL con `admin`. Se elimina `devRootHosts`.)*
- `<slug>.<rootDomain>` / `<slug>.localhost` con slug no reservado → **TENANT**.
- Dominio raíz, slugs reservados distintos de `admin`, y **cualquier otro host** → nuevo estado **UNRECOGNIZED**: el login muestra un aviso ("Esta dirección no corresponde a ningún acceso") sin formulario y el SPA no manda `X-Tenant-Host`.

El interceptor manda `X-Tenant-Host: <slug>` en TENANT y `X-Tenant-Host: admin` en SUPER_ADMIN (el valor sale de `environment.platformSlug`); nada en UNRECOGNIZED. `tenantDomainGuard` pasa a exigir TENANT (hoy dejaría pasar cualquier no-SuperAdmin, lo que con un tercer estado abriría el dashboard de negocio sin negocio).

**Rationale**: es lo que obliga FR-006 (el dominio raíz no ofrece login) y evita la rama silenciosa "host desconocido ⇒ Super Admin" de H-3. Las rutas públicas del comensal (`menu/t/:token`) no dependen del contexto (el token firmado lleva el negocio), así que siguen funcionando en cualquier host. **Alternatives**: dejar UNRECOGNIZED como SUPER_ADMIN y que solo el backend rechace → el SPA mandaría `admin` desde la raíz y el backend lo aceptaría (violaría FR-006). Redirigir la raíz a `admin.` → la raíz sirve otra página y no pasa por este SPA.

### D5 — Reglas del nombre completo: una sola definición, dos implementaciones con vectores compartidos

**Decision**: orden de evaluación **obligatorio → longitud → formato** (FR-015). Tras `trim` y normalización **NFC** (para que "María" escrito con tilde combinada cuente igual): 2–100 caracteres (por **puntos de código**), solo letras latinas, espacio, apóstrofe (`'` y `’`) y guion (`-`), con **al menos dos letras**. "Letra latina" = rangos explícitos `A-Za-z`, `À-Ö`, `Ø-ö`, `ø-ÿ` (excluye × y ÷), `Ā-ɏ` (Latinas Extendidas A/B) y `Ḁ-ỿ` (Latinas Extendidas Adicionales: vietnamita, etc.). Rechaza dígitos, puntuación, emoji, tab/salto de línea, otros alfabetos y cualquier `<`/`>`. Definición y **tabla de vectores de prueba** en [contracts/full-name-rules.md](./contracts/full-name-rules.md); ambos repos implementan la misma tabla en sus pruebas.

**Rationale**: la lista blanca de caracteres es la defensa anti-XSS de FR-017 (si `<` y `>` no pasan, no hay etiquetas); el escape en el correo (D7) es la segunda capa. Rangos explícitos en vez de `\p{L}` porque `re` de Python no los soporta y `\p{Script=Latin}` en JS incluye símbolos (ª, º) que la spec no acepta; así ambos motores coinciden. **Alternatives**: librería `regex` (Python) → dependencia nueva sin necesidad (Principio IX); validar solo en el cliente → incumple FR-016.

### D6 — Persistencia: columna nullable en la invitación, sin backfill

**Decision**: `user_invitations.name VARCHAR(100) NULL` (migración nueva sobre `a89f0c1d2e3b`). Al consumir: `User.name = invitation.name or invitation.email` (conserva el comportamiento legacy para invitaciones anteriores, FR-024). Reenviar no toca el nombre (FR-018). `InvitationResponse.name` es opcional. Detalle y rollback en [data-model.md](./data-model.md).

**Rationale**: aditivo, sin default y sin reescribir filas → no cambia el significado histórico de ningún dato (Principio VIII) y no toca facturas (VII). **Alternatives**: `NOT NULL` con backfill al correo → inventa un valor y rompe el "(o solo el correo si es una invitación anterior)" de la lista de pendientes; guardar el nombre en el `User` al invitar (crear la cuenta antes) → contradice el diseño de spec 037 (la cuenta nace al primer login).

### D7 — Campo `name` obligatorio en el servicio, incluso si el cliente lo omite

**Decision**: `InvitationCreate.name` se declara `Field(default=None, validate_default=True)` con un validador que corre siempre, de modo que **omitir la clave** produce 422 con el mismo mensaje "El nombre es obligatorio" (FR-016, Escenario 7) en vez de un genérico "Field required". El mensaje queda en `detail[0].msg` con el prefijo estándar de Pydantic (`Value error, …`). El frontend, que hoy asume `detail: string`, aprende a leer el arreglo de 422 y quita el prefijo.

**Rationale**: un único mensaje para vacío/solo espacios/ausente, igual que en pantalla. **Alternatives**: `name: str` requerido a secas → mensaje distinto para "ausente"; handler global de 422 que reescriba mensajes → toca todos los endpoints (Principio V).

### D8 — Correo: saludo con el nombre, escapado

**Decision**: `invitation_email_body(..., name: str | None = None)` antepone `Hola, <nombre>:` (nombre pasado por `html.escape(..., quote=True)`) o `Hola:` si no hay nombre (invitación legacy, FR-019). El resto del cuerpo no cambia. El asunto no cambia.

**Rationale**: con la lista blanca de D5 el escape no debería tener qué escapar salvo el apóstrofe (`O'Brien` → `O&#x27;Brien`, que el cliente de correo muestra como `O'Brien`); se mantiene por defensa en profundidad y porque H-9 muestra que hoy no hay escape alguno. **No** se escapan en esta spec `tenant_name`/`email`/`password` (Principio V; queda anotado como deuda aparte).

### D9 — Lista de usuarios y avatar: un único helper de presentación

**Decision**: `displayName(name, email) = name?.trim() || email` y `initialOf(...)` (primer **punto de código**, en mayúscula) en un helper compartido del frontend, usado por la página "Usuarios", la lista de invitaciones pendientes y la tabla del Super Admin (FR-020–FR-022). Nada se migra: "nombre = correo" (legacy real, H-8) se ve como correo sin tocar datos. El título de fila usa `truncate` (ya presente) y el contenedor `min-w-0` para que 100 caracteres no rompan el diseño (Historia 5.4).

### D10 — Limpieza del formulario (Escenario 9, FR-023)

**Decision**: el formulario vive bajo `@if (showForm())`, así que sus campos se recrean vacíos, **pero** el mensaje de error del formulario es un `signal` en `InvitationsService` (singleton) y **sobrevive** a cerrar/abrir. Se limpia `invitationsService.error` y se hace `form.reset()` al cancelar, tras un envío exitoso y al iniciar el componente. Cubre campos y mensajes (FR-023).

### D11 — Historia 6 (formulario del Super Admin): decisión abierta O-1

**Hecho (H-10)**: el formulario llama a endpoints que no existen. **Opciones**:

| Opción | Efecto | Costo |
|---|---|---|
| **A (recomendada, adoptada por defecto en este plan)** | Aplicar al formulario el mismo validador (nombre obligatorio al crear; al editar, opcional pero válido si se escribe — Historia 6.4) y el helper de presentación a su tabla. **Sin endpoints nuevos.** | Mínimo; cumple FR-012/FR-020 en pantalla. El guardado seguirá fallando (405) hasta que exista el endpoint, y eso queda **registrado** (A-102), no escondido. |
| B | Retirar "Nuevo usuario"/"Editar" de la pantalla | Es un cambio de producto distinto (spec propia). |
| C | Crear `POST/PATCH /super-admin/users` | Funcionalidad nueva y contraria a la decisión de spec 037 (alta solo por invitación); spec propia. |

**Consecuencia para la verificación**: el criterio 6.3 ("la tabla lo muestra con ese nombre") solo puede probarse con usuarios **ya existentes** (nombre mostrado + inicial), no con una alta de punta a punta. Se pide al dueño de la spec confirmar A o elegir B/C; el plan aísla esta historia (Incremento 3) para que cambiarla no afecte a los otros dos.

### D12 — Despliegue y compatibilidad

- **Orden**: (1) verificación previa en BD: ningún negocio con `host` reservado (consulta en [quickstart.md](./quickstart.md)); (2) migración; (3) backend; (4) frontend. Entre (3) y (4) un SPA antiguo (en caché por el service worker) mandará invitaciones sin nombre → 422 con el mensaje "El nombre es obligatorio"; es el costo aceptado de exigir el campo en el servicio (FR-016) y se mitiga desplegando 3 y 4 seguidos. Un SPA antiguo en `admin.` seguiría sin mandar `admin` → login 401 hasta recargar la nueva versión.
- **Sesiones abiertas** del Super Admin desde el dominio raíz u otros hosts: el token sigue válido en el API, pero el SPA ya no lo trata como plataforma en esos hosts (UNRECOGNIZED); entra de nuevo por `admin.` sin perder datos (Edge Case de la spec).
- **Rollback**: revertir frontend y backend; la migración se revierte con `downgrade` (borra solo `user_invitations.name`; efecto: las invitaciones pendientes pierden el nombre capturado, no la invitación).
- **Dependencias nuevas**: ninguna.

### D13 — Anomalías a registrar **antes** de implementar (Principios II y XI)

Numeración tentativa (la siguiente libre tras A-98; se confirma al registrar): **A-99** el login de plataforma exige el host `admin` y ya no acepta Super Admin sin cabecera ni desde hosts desconocidos/raíz; **A-100** crear una invitación exige nombre completo validado (antes no existía el campo); **A-101** reservados `admin`, `assets`, `api`, `docs` además de `www`, `app`; **A-102** hallazgo H-10 (formulario del Super Admin apunta a endpoints inexistentes) y la decisión O-1. Quién y cuándo: el dueño de la spec, 2026-10-02 (clarificaciones de la spec).
