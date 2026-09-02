# Research: Manejo de excepciones y respuestas de error consistentes en el módulo super-admin

Auditoría realizada directamente sobre `../pos-backend` (commit de trabajo actual en `main`),
antes de diseñar. Cada decisión cita el archivo/línea que la respalda.

## 1. Estado actual del manejo de errores en `pos-backend`

- **No existe ningún manejador de excepciones global.** `app/main.py` no registra ningún
  `@app.exception_handler(...)`. Toda excepción no atrapada llega al manejador por defecto de
  Starlette (`HTTPException` → `{"detail": ...}`; excepción no manejada → 500 sin cuerpo JSON
  estructurado, potencialmente con traza si el proceso corre en modo debug).
- **No existe `request_id` en ninguna respuesta hoy** (`grep -rn "request_id" app` → 0
  resultados fuera de este spec).
- **`sentry-sdk==2.61.0` está en `requirements.txt` pero no se usa en ningún archivo de `app/`**
  (`grep -ril sentry app` → solo `requirements.txt`). No hace falta justificar una dependencia
  nueva (Principio IX): solo integrarla.
- **`app/core/exceptions.py`** ya tiene 5 excepciones, todas de autenticación
  (`InvalidToken`, `RevokedToken`, `AccessTokenRequired`, `RefreshTokenRequired`, `UserNotFound`)
  más `InsufficientStockError` (subclase de `HTTPException`, usada por el módulo de inventario) y
  `TenantNotFoundError` (una excepción de dominio ya "correcta" en el sentido de no depender de
  FastAPI — pero **no se usa en ningún punto del código actual**, `grep -rn TenantNotFoundError
  app` solo la encuentra en su propia definición). Ninguna de las cinco excepciones de auth se
  usa dentro de super-admin; no se tocan.

**Decisión**: no reutilizar `app/core/exceptions.py` como destino de las nuevas excepciones de
dominio de super-admin. Es el archivo de excepciones de autenticación del proyecto, ya usado
fuera de este módulo, y mezclar ahí una jerarquía de dominio genérica sería forzar una
responsabilidad que no tiene. En su lugar: `app/core/domain_errors.py`, nuevo, sin ningún import
de FastAPI/Starlette. `TenantNotFoundError` se dedeja intacta donde está (no se elimina: podría
estar reservada para el flujo de resolución de tenant por host, aunque hoy no se use; tocarla no
es necesario para este spec y sería un cambio no relacionado).

## 2. El patrón de errores que super-admin ya reutiliza (y que este spec no debe duplicar)

Los tres archivos del módulo (`router.py`, `plans_router.py`, `payment_methods_router.py`, 529
líneas) **no tienen ni un solo `raise HTTPException(...)` inline propio**. Todo error de negocio
pasa por tres funciones compartidas, usadas también por otros módulos:

| Helper | Ubicación | Usado también por | Qué lanza hoy |
|---|---|---|---|
| `get_or_404(db, model, id, detail)` | `app/core/crud.py` | 24 archivos en 15+ módulos | `HTTPException(404, detail)` |
| `ensure_unique(db, model, field, value, detail, exclude_id)` | `app/core/crud.py` | (mismo archivo, mismo uso compartido) | `HTTPException(409, detail)` |
| `validate_billing_cycle_price(plan, ciclo)` | `app/core/plan_limits.py` | 19 archivos en 12+ módulos | `HTTPException(409, mensaje con nombre del plan/ciclo)` |

**Decisión**: no modificar ninguna de las tres. Tocarlas violaría directamente la restricción del
usuario ("NO dupliques infraestructura global existente", "NO migres otros módulos") porque
cualquier cambio de comportamiento en ellas se propagaría de inmediato a los otros 20+ módulos
que las llaman. El pedido original de "domain-ificar" cada punto de fallo del módulo choca aquí
con la realidad: **no hay puntos de fallo propios del módulo que domain-ificar** — todos delegan
en helpers compartidos ya funcionales. Se documenta como hallazgo (Principio V: no forzar un
refactor que el spec no exige).

`get_current_super_admin` (`app/core/dependencies.py`) sí es exclusiva de este módulo (nada más
la usa), pero también vive en un archivo compartido con las dependencias de autenticación de
tenant. Se deja intacta: ya produce exactamente el código HTTP correcto (403 sin rol de
super-admin) y no está protegida contra un cambio de forma en su cuerpo de error — el envelope
nuevo la cubre igual que a cualquier otro `HTTPException`, sin tocar su código.

## 3. Cómo obtener `tenant_id`/`user_id` hoy — y por qué "aislamiento multi-tenant" significa algo
   distinto en este módulo

`get_current_super_admin` (`app/core/dependencies.py:192`) no depende de un `x-tenant-host`: lee
el JWT (`get_valid_token_data`), exige el claim `is_super_admin`, y busca al usuario por
`email` **con `User.tenant_id.is_(None)`** — es decir, el actor autenticado de este módulo es un
usuario global, sin tenant propio. Esto está confirmado también en el modelo (`app/core/models.py`):
`Tenant` no tiene ningún campo de estado activo/inactivo, así que "tenant inactivo" (mencionado en
el pedido original) no es un caso representable hoy en el esquema.

**Decisión** (ya reflejada en `spec.md` § Assumptions): el aislamiento multi-tenant de este
módulo no es "un tenant no debe ver los datos de otro tenant" (ese modelo no aplica: el trabajo
del super-admin es administrar a todos los tenants). Es, en cambio: (a) que la autorización siga
basada exclusivamente en el rol del token, nunca en un `tenant_id` de body/query/path (ya lo
hace `get_current_super_admin`; no se toca), y (b) que cualquier `tenant_id`/`plan_id`/`catalog_id`
recibido en la ruta o el cuerpo se use solo como clave de búsqueda validada — exactamente lo que
`get_or_404` ya hace hoy.

## 4. Consumidor existente del contrato de error: panel de super-admin en `pos-heladeria`

`grep -rn "err.error" ../pos-heladeria/src/app/modules/super-admin` confirma 4 servicios
Angular (`tenant.service.ts`, `plan.service.ts`, `payment-method-catalog.service.ts`,
`super-admin-users.service.ts`) que leen `err.error.detail` como texto plano; uno de ellos cae a
`err.error.message` si falta. Ninguno espera hoy los campos `success`/`error.code`/`request_id`.

**Decisión** (ya registrada en `spec.md` § Clarifications, confirmada con el usuario): el
envelope nuevo agrega esos campos pero **conserva** un `detail` de nivel superior con el mismo
texto de `error.message`, así los 4 servicios siguen funcionando sin tocar `pos-heladeria`. Fuera
de alcance de este spec: migrar esos servicios a leer los campos estructurados.

## 5. Characterization tests existentes del módulo — qué protegen realmente

`app/characterization_tests/test_super_admin_plans.py` y
`test_super_admin_payment_catalog.py` (ambos con nota explícita: *"`Depends(get_current_super_admin)`
vive en el router padre"* / *"solo se aplica en el router padre"*) invocan las funciones del
router **directamente**, sin `TestClient` ni la app ASGI completa, y aseveran con
`assertRaises(HTTPException)` (incluyendo `.status_code == 409`) sobre los `raise` que ocurren
dentro de `ensure_unique`/`get_or_404`.

**Implicación de diseño, verificada, no solo asumida**: como estos tests llaman a las funciones
del router en proceso (no a través de la app), **cualquier middleware ASGI registrado en
`app/main.py` es completamente invisible para ellos** — el middleware solo intercepta
solicitudes reales servidas por la app. Mientras `ensure_unique`/`get_or_404` sigan intactas
(decisión de la sección 2), estos characterization tests permanecen en verde sin ningún cambio,
cumpliendo el Principio III sin necesidad de "autorizar" nada porque no hay comportamiento
protegido que cambie.

**Otro hallazgo, fuera de alcance**: `validate_billing_cycle_price` (`app/core/plan_limits.py:184`)
tiene una rama `raise ValueError(...)` para un `ciclo_facturacion` inválido que, en el endpoint
`PATCH /super-admin/tenants/{tenant_id}`, es hoy inalcanzable — el schema `TenantPlanUpdate`
(`app/api/v1/super_admin/schemas.py:95`) ya restringe `ciclo_facturacion` a
`Literal["mensual", "anual"]`, así que Pydantic rechaza cualquier otro valor con 422 antes de
llegar a esa función. Se documenta como hallazgo, no se modifica `plan_limits.py` (compartido);
si en algún punto no cubierto por esta restricción ese `ValueError` sí llegara a ejecutarse, cae
dentro del camino genérico de "falla técnica inesperada" (sección 7) y ya no produce una traza
cruda al llamador.

## 6. Entorno de ejecución y gate de Sentry

`app/core/config.py:113` documenta la intención original en su propio comentario: *"Ambiente de
ejecución: 'prod' o 'dev'"*. `grep -rn 'ENVIRONMENT ==' app` confirma que las tres comparaciones
existentes en el código (`auth/routes.py`, `invitations/router.py`, `super_admin/router.py`)
comparan siempre contra el literal `"prod"`; no hay un tercer/cuarto valor (`"test"`/`"staging"`)
en uso en `.env`/`.env.example`/`docker-compose*.yml`.

**Decisión**: reutilizar exactamente ese mismo criterio (`settings.ENVIRONMENT == "prod"`) para
decidir cuándo inicializar Sentry, en vez de introducir nuevos valores de entorno que el resto
del proyecto no usa. Si en el futuro se agregan `"test"`/`"staging"` como valores reales, seguirán
cayendo del lado de "no producción" sin ningún cambio adicional, porque el gate es
`== "prod"`, no `!= "dev"`.

**Decisión sobre el DSN**: un único `SENTRY_DSN: Optional[str]` nuevo en `Settings`
(`app/core/config.py`), leído del entorno igual que el resto de la configuración. `sentry_sdk.init()`
se llama **una sola vez**, en el arranque de la app (`create_app()`/`lifespan`), solo si
`ENVIRONMENT == "prod"` **y** `SENTRY_DSN` está presente. Fuera de producción, `sentry_sdk.init()`
nunca se llama — el SDK, sin inicializar, hace que cualquier `capture_exception`/`capture_message`
sea un no-op silencioso (comportamiento documentado del propio SDK). Esto cumple FR-010/FR-011
sin necesitar comprobar el entorno en cada punto donde se captura un error: basta con no
inicializar fuera de producción.

## 7. Diseño del punto de traducción a HTTP: middleware con alcance de prefijo, no
   `@app.exception_handler`

Se evaluaron dos formas de traducir excepciones a respuestas HTTP:

- **Handler global (`@app.exception_handler(Exception)` / `(HTTPException)`)**: es el patrón
  idiomático de FastAPI, pero un handler así es *inherentemente global* — se aplicaría a las 23+
  rutas de los otros módulos también, cambiando su forma de respuesta (para `HTTPException`) o su
  comportamiento ante fallas no manejadas (para `Exception`) sin que nadie haya tocado su código.
  Esto viola directamente la restricción "únicamente en el módulo super-admin" del pedido
  original, incluso sin escribir una sola línea en otro módulo.
- **Middleware ASGI parametrizado por prefijo de ruta (elegido)**: un
  `BaseHTTPMiddleware`/middleware ASGI puro que, en `app/main.py`, se registra con
  `path_prefix="/api/v1/super-admin"`. Envuelve `call_next`; si la ruta de la solicitud no
  empieza por ese prefijo, no hace nada distinto de hoy (pasa la solicitud/excepción tal cual).
  Si sí empieza por ese prefijo, atrapa `domain_errors.*`, `HTTPException` (venga de
  `get_or_404`/`ensure_unique`/`validate_billing_cycle_price` o de cualquier otro origen) y
  cualquier `Exception` no anticipada, y construye el envelope consistente para las tres. Es la
  **única** pieza nueva verdaderamente global en el árbol de rutas, y su parametrización por
  prefijo es exactamente el mecanismo documentado para que, en el futuro, migrar otro módulo sea
  agregar su prefijo a la lista — sin duplicar el middleware.

**Decisión**: middleware parametrizado por prefijo. Vive en `app/core/error_middleware.py`
(reutilizable) y se instancia en `app/main.py` con el prefijo de super-admin únicamente.

## 8. Clasificación de errores de negocio → código estable de `error.code`

Como ningún punto de fallo propio del módulo necesita cambiar (sección 2), el middleware deriva
`error.code` a partir del **código de estado HTTP** que ya produce cada camino existente, no de
un código por entidad hardcodeado en cada `raise`:

| Código HTTP | `error.code` | Origen típico en este módulo |
|---|---|---|
| 401 | `UNAUTHORIZED` | `get_valid_token_data` (token inválido/ausente) |
| 403 | `FORBIDDEN` | `get_current_super_admin` (rol distinto de super-admin) |
| 404 | `NOT_FOUND` | `get_or_404` (tenant/plan/catálogo inexistente) |
| 409 | `CONFLICT` | `ensure_unique`, `validate_billing_cycle_price` |
| 422 | `INVALID_INPUT` | Validación de Pydantic (FastAPI, sin cambios) |
| 500 | `INTERNAL_ERROR` | Cualquier `Exception` no anticipada |

El mensaje específico a la entidad (que FR-005 exige) ya lo provee, hoy, el `detail` que cada
llamada a `get_or_404`/`ensure_unique` pasa explícitamente (p. ej. *"No existe ese tenant"*, *"No
existe ese plan"*) — se preserva tal cual como `error.message`/`detail`. Las excepciones nuevas de
`domain_errors.py` (para el código, si alguno, que en el futuro se escriba directamente en el
módulo en vez de delegar en los helpers compartidos) sí permiten declarar un `code` más
específico por entidad (p. ej. `TENANT_NOT_FOUND`); el middleware usa ese código cuando la
excepción lo trae, y cae al mapeo genérico de la tabla anterior cuando no.

**Alternativa considerada y descartada**: reemplazar cada llamada a `get_or_404`/`ensure_unique`
dentro de super-admin por wrappers locales que lancen excepciones de dominio con código específico
por entidad (`TENANT_NOT_FOUND`, `PLAN_NOT_FOUND`, `PAYMENT_METHOD_CATALOG_ENTRY_NOT_FOUND`, ...).
Se descartó para este spec: no aporta nada verificable que el mensaje específico ya existente no
dé, y multiplica superficie de cambio (y por tanto riesgo sobre los characterization tests) sin
necesidad. Queda documentado en Assumptions/quickstart como extensión natural, no bloqueante, si
en el futuro un consumidor necesita distinguir por código en vez de por mensaje.

## 9. El caso del correo de bienvenida (`create_tenant`) — falla técnica tolerada, no error de negocio

`router.py:143-161`: si el envío del correo de bienvenida falla (Redis/worker caído), la excepción
se atrapa localmente, se registra con `logger.warning(..., exc_info=True)`, y la respuesta sigue
siendo éxito — el tenant ya fue creado y commiteado. Esto **no pasa nunca** por el middleware
(la excepción no se propaga).

**Decisión**: preservar el comportamiento exacto de cara a quien llama (FR-007) y agregar, dentro
de ese mismo bloque `except`, una llamada a `sentry_sdk.capture_exception()` (no-op fuera de
producción por el gate de la sección 6) para que el equipo técnico vea en producción que la
mensajería de bienvenida está fallando, sin que esa visibilidad afecte la respuesta.

## 10. Verificación

Los characterization tests existentes de super-admin no cambian y deben seguir pasando
exactamente como hoy (sección 5). Se agrega un archivo de tests nuevo
(`test_super_admin_error_envelope.py`) usando `fastapi.testclient.TestClient` — el único patrón
que atraviesa el middleware real — para verificar, contra la app completa: la forma del envelope
en 404/409/403/401/422/500, que el campo `detail` sigue presente, que no hay Sentry activo cuando
`ENVIRONMENT != "prod"` (se puede verificar interceptando/mockeando `sentry_sdk.capture_exception`
o comprobando que `sentry_sdk.init` no fue llamado), y que las rutas de otros módulos (una de
control, ya existente) no cambian su forma de respuesta de error.

## 11. Correcciones aplicadas durante la implementación (no durante la planeación)

Dos ajustes descubiertos al escribir el código real, ninguno cambia las decisiones de fondo de
las secciones 1-10, pero corrigen afirmaciones inexactas que sí quedaron escritas en `plan.md`/
`tasks.md` durante `/speckit-plan`:

- **`TestClient` no es un patrón ya usado en este repo.** `plan.md`/`tasks.md` afirmaban que
  `test_invitations_*.py` ya lo usaba; al escribir los tests se confirmó lo contrario
  (`grep -rn "TestClient(" app` → 0 resultados) — esos archivos dicen explícitamente en su
  docstring que **no** lo usan. Es, por tanto, un patrón nuevo para este repositorio (necesario:
  es la única forma de ejercitar middleware/`exception_handler` ASGI reales), sin que eso
  implique una dependencia nueva (`httpx` ya está en `requirements.txt`, es lo que `TestClient`
  necesita). Corregido en `plan.md` § Technical Context y `tasks.md` T010.
- **El mecanismo de traducción a HTTP no es un único `BaseHTTPMiddleware` que envuelve y
  reescribe cualquier respuesta.** El diseño original (sección 7, arriba) subestimó cómo Starlette
  ordena su stack: `HTTPException`/`RequestValidationError` ya se convierten en una `Response`
  dentro de `ExceptionMiddleware`, que corre **dentro** de cualquier middleware de usuario — para
  cuando la excepción llegaría a un `try/except` en un `BaseHTTPMiddleware`, ya no es una
  excepción, es una respuesta hecha (`{"detail": ...}`), y reescribirla exige consumir y
  reconstruir su `body_iterator` a mano. Peor aún: un `@app.exception_handler(Exception)` global
  no se registra en `ExceptionMiddleware` sino en `ServerErrorMiddleware` (el más externo de
  todos) — delegar ahí "para otras rutas" con un `raise exc` deja la solicitud sin respuesta
  alguna (regresión real para otros módulos, exactamente lo que este spec prohíbe).

  Diseño implementado en su lugar (`app/core/error_middleware.py`, `../pos-backend`):
  - `RequestIdMiddleware`: solo estampa `request.state.request_id`, y **solo para el prefijo
    `/api/v1/super-admin`** envuelve `call_next` en un `try/except Exception` — para cualquier
    otra ruta no hace nada (ni genera id ni entra al `try/except`), así que su comportamiento por
    unhandled exceptions es exactamente el de hoy, sin tocar `ServerErrorMiddleware`.
  - `register_error_handlers`: tres `@app.exception_handler` (`HTTPException`,
    `RequestValidationError`, `DomainError`) — los tres se resuelven dentro de
    `ExceptionMiddleware`, nunca en `ServerErrorMiddleware`, así que delegar en el handler por
    defecto de FastAPI/Starlette para rutas fuera del prefijo es seguro y no deja ninguna
    solicitud sin respuesta.
  - Nunca se registra `@app.exception_handler(Exception)`: el caso "excepción no anticipada" para
    super-admin lo cubre el `try/except` de `RequestIdMiddleware`, ya scoped por prefijo desde el
    primer `if`.

  Sigue siendo, en espíritu, exactamente lo que `plan.md`/`data-model.md`/`contracts/` describen
  (traducción a HTTP con alcance de prefijo, sin tocar otros módulos) — solo cambia, a nivel de
  implementación, *cómo* se logra ese alcance dentro de las capas de Starlette.
