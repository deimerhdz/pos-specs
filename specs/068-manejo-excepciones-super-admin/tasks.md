---

description: "Task list template for feature implementation"
---

# Tasks: Manejo de excepciones y respuestas de error consistentes en el módulo super-admin

**Input**: Documentos de diseño de `/specs/068-manejo-excepciones-super-admin/`
**Repositorio de implementación**: `../pos-backend` (todas las rutas de archivo de abajo son
relativas a ese repositorio hermano, no a `pos-specs`)
**Prerequisites**: plan.md, spec.md, research.md, data-model.md, contracts/error-envelope.md,
quickstart.md

**Tests**: `spec.md` exige verificación explícita (FR-001 a FR-015, criterios de aceptación de
las 3 historias) y la constitución del proyecto (Principio X) hace la verificación obligatoria,
así que este plan sí incluye tareas de test.

**Organization**: Las tareas están agrupadas por historia de usuario (US1/US2/US3, en el orden de
prioridad de `spec.md`) para poder implementar y verificar cada una por separado.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Puede ejecutarse en paralelo (archivo distinto, sin dependencias pendientes)
- **[Story]**: A qué historia de usuario pertenece (US1, US2, US3)
- Cada tarea incluye la ruta de archivo exacta

## Path Conventions

Proyecto único (backend FastAPI). Todas las rutas son relativas a `../pos-backend/` desde este
repositorio de specs.

---

## Phase 1: Setup

**Purpose**: Crear los archivos nuevos que la Fase 2 va a llenar, y el campo de configuración que
necesita la integración con Sentry. Ningún archivo compartido con otros módulos se toca aquí.

- [X] T001 [P] Agregar el campo `SENTRY_DSN: Optional[str] = Field(default=None, env="SENTRY_DSN")`
  a la clase `Settings` en `../pos-backend/app/core/config.py` (`data-model.md` § 4)
- [X] T002 [P] Crear el archivo `../pos-backend/app/core/domain_errors.py` (vacío salvo docstring
  de módulo — **sin ningún import de `fastapi`/`starlette`**, por diseño: `data-model.md` § 1)
- [X] T003 [P] Crear el archivo `../pos-backend/app/core/error_response.py` (vacío salvo docstring
  de módulo)
- [X] T004 [P] Crear el archivo `../pos-backend/app/core/error_middleware.py` (vacío salvo
  docstring de módulo)

**Checkpoint**: Archivos base creados; ninguno tiene aún comportamiento.

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: Construir la jerarquía de excepciones de dominio, el envelope de error y el
middleware ASGI con alcance de prefijo de ruta que las tres historias de usuario necesitan para
poder verificarse. Nada de esta fase toca `app/core/crud.py`, `app/core/plan_limits.py`, ni
ninguna función de `app/core/dependencies.py` distinta de `get_current_super_admin`
(`research.md` § 2, `plan.md` § Constraints).

**⚠️ CRITICAL**: Ninguna historia de usuario puede verificarse hasta que esta fase esté completa.

- [X] T005 Implementar la jerarquía de excepciones en
  `../pos-backend/app/core/domain_errors.py`: clase base `DomainError(code, message,
  details=None)` y subclases `EntityNotFoundError`, `ConflictError`, `InvalidStateError`,
  `BusinessRuleViolation`, `UnauthorizedError`, `ForbiddenError`, cada una con el código HTTP por
  defecto que le corresponde según `data-model.md` § 1. Depende de: T002.
- [X] T006 Implementar en `../pos-backend/app/core/error_response.py` la función que construye el
  envelope (`success`, `error.code`, `error.message`, `error.details`, `request_id`, y el campo de
  compatibilidad `detail` igual a `error.message` — `spec.md` § Clarifications, `data-model.md` §
  2) a partir de: una `DomainError`, o un `(status_code, detail)` de `HTTPException`, o una
  `Exception` genérica. Incluir la tabla de mapeo código HTTP → `error.code` por defecto de
  `research.md` § 8 (401→`UNAUTHORIZED`, 403→`FORBIDDEN`, 404→`NOT_FOUND`, 409→`CONFLICT`,
  422→`INVALID_INPUT`, 500→`INTERNAL_ERROR`), y un mensaje genérico seguro y fijo para el caso
  500 que **nunca** incluye el texto de la excepción original. Depende de: T003, T005.
- [X] T007 Implementar en `../pos-backend/app/core/error_middleware.py` un middleware ASGI
  (`BaseHTTPMiddleware` o middleware ASGI puro) parametrizado por `path_prefix: str`: genera un
  `request_id` (UUID4) al inicio de cada solicitud y lo guarda en `request.state.request_id`;
  para solicitudes cuya ruta no empiece por `path_prefix`, no cambia nada del comportamiento
  actual; para las que sí, envuelve la solicitud en un `try/except` que captura, en este orden,
  `DomainError`, `HTTPException` y `Exception`, construye la respuesta con
  `error_response.py` (T006), y para el caso `Exception` registra el error del lado del servidor
  con `logger.exception(...)` (incluyendo el `request_id`) sin que ese detalle llegue nunca al
  cuerpo de la respuesta (FR-003). Depende de: T004, T006.
- [X] T008 Registrar el middleware en `create_app()` de `../pos-backend/app/main.py` con
  `path_prefix="/api/v1/super-admin"` (único prefijo activado por este spec). Depende de: T007.
- [X] T009 En `../pos-backend/app/main.py` (junto al resto del arranque en `create_app()` o
  `lifespan`), llamar `sentry_sdk.init(dsn=settings.SENTRY_DSN, environment=settings.ENVIRONMENT,
  ...)` **solo si** `settings.ENVIRONMENT == "prod"` **y** `settings.SENTRY_DSN` no es `None`
  (`research.md` § 6). Depende de: T001, T008 (mismo archivo).

**Checkpoint**: El middleware está activo solo para `/api/v1/super-admin`; Sentry solo se
inicializa en producción y con DSN configurado. Ninguna otra ruta del backend cambió de
comportamiento.

---

## Phase 3: User Story 1 - Errores de negocio claros y sin fugas de información técnica (Priority: P1) 🎯 MVP

**Goal**: Toda solicitud fallida a un endpoint de super-admin devuelve el envelope consistente,
con el código HTTP correcto y sin ningún detalle técnico interno, sin importar si el motivo es
un error de negocio esperado o una falla técnica inesperada.

**Independent Test**: Contra la app real (vía `TestClient`), invocar cada endpoint del módulo con
datos que disparan cada motivo de fallo conocido (recurso inexistente, conflicto, sin permiso,
sin autenticar, falla inesperada) y verificar la forma del envelope, el código HTTP, y la ausencia
de SQL/trazas/credenciales/tokens en el cuerpo; verificar además que una ruta de control fuera del
módulo no cambió su forma de respuesta.

### Tests for User Story 1

- [X] T010 [US1] Escribir, en
  `../pos-backend/app/characterization_tests/test_super_admin_error_envelope.py` (nuevo, vía
  `starlette.testclient.TestClient` — patrón nuevo para este repo, no usado hoy en ningún test
  existente (`plan.md` § Testing); no agrega dependencias, `httpx` ya está en
  `requirements.txt`), un
  caso por cada fila de la tabla del paso 2 de `quickstart.md`: tenant inexistente (404),
  plan inexistente (404), ciclo de facturación sin precio (409), nombre de plan duplicado (409),
  sin rol de super-admin (403), sin token/token inválido (401), falla técnica inesperada forzada
  (500). Cada caso verifica: `success == false`, `error.code` según la tabla de `research.md` §
  8, `error.message`/`detail` presentes y sin fragmentos de SQL/trazas/rutas de archivo/tokens, y
  `request_id` con forma de UUID. Depende de: T008.
- [X] T011 [P] [US1] Escribir, en
  `../pos-backend/app/characterization_tests/test_error_middleware_scope.py` (nuevo), un caso de
  control: invocar un endpoint de otro módulo con un error conocido (p. ej. `GET
  /api/v1/products/{uuid-inventado}`, que hoy responde 404 con `{"detail": "..."}` plano) y
  verificar que la respuesta **no** trae los campos `success`/`error`/`request_id` — confirma que
  el middleware de super-admin no se activó fuera de su prefijo. Depende de: T008.
- [X] T012 [US1] Ejecutar
  `python -m unittest app.characterization_tests.test_super_admin_plans -v` y
  `python -m unittest app.characterization_tests.test_super_admin_payment_catalog -v` en
  `../pos-backend` y confirmar que ambos siguen en verde, sin ninguna modificación a esos dos
  archivos (`research.md` § 5 — corren en proceso, sin pasar por el middleware). Depende de: T007.

### Implementation for User Story 1

- [X] T013 [US1] Revisar `../pos-backend/app/core/error_middleware.py` contra los resultados de
  T010/T011: ajustar el mapeo de `error.code`, el mensaje genérico del caso 500, o el orden de
  captura de excepciones hasta que todos los casos de T010 y T011 pasen. Depende de: T010, T011,
  T012.

**Checkpoint**: User Story 1 es funcional y verificable de forma independiente — el módulo
super-admin ya responde con el envelope consistente y seguro para todo su catálogo de errores
conocido.

---

## Phase 4: User Story 2 - Observabilidad en producción sin ruido de errores esperados (Priority: P2)

**Goal**: En producción, toda falla técnica inesperada del módulo queda registrada en Sentry con
`request_id`/`user_id`/módulo/operación; ningún error de negocio esperado genera un evento; fuera
de producción no se envía nada; el módulo sigue funcionando con Sentry deshabilitado.

**Independent Test**: Con `ENVIRONMENT=prod` y `sentry_sdk.capture_exception` interceptado,
provocar una falla técnica inesperada y verificar exactamente una llamada con el contexto
esperado; provocar errores de negocio esperados (404/409/403/401) y verificar cero llamadas; con
`ENVIRONMENT=dev`, verificar cero llamadas para cualquier tipo de error y que `sentry_sdk.init`
nunca se invocó.

### Tests for User Story 2

- [X] T014 [US2] Escribir, en
  `../pos-backend/app/characterization_tests/test_super_admin_sentry_integration.py` (nuevo),
  los casos: (a) con `ENVIRONMENT="prod"` y `SENTRY_DSN` de prueba, mockeando
  `sentry_sdk.capture_exception`, forzar una falla técnica inesperada en un endpoint de
  super-admin y verificar una única llamada con `request_id`, `user_id` (o `None` si la solicitud
  no llegó a autenticarse), `module="super-admin"` y `operation` en las tags/contexto (FR-008,
  `data-model.md` § 3); (b) en el mismo entorno, provocar cada error de negocio esperado de T010 y
  verificar cero llamadas a `capture_exception` (FR-009); (c) con `ENVIRONMENT="dev"` (valor por
  defecto del repo), repetir (a) y (b) y verificar cero llamadas en ambos casos, y que
  `sentry_sdk.Hub.current.client` es `None` (Sentry nunca se inicializó, FR-010); (d) con
  `SENTRY_DSN` sin configurar (`None`) y `ENVIRONMENT="prod"`, verificar que el módulo responde
  con normalidad a una solicitud fallida y que no hay ninguna llamada a `capture_exception`
  (FR-011). Depende de: T009.

### Implementation for User Story 2

- [X] T015 [US2] En `../pos-backend/app/core/dependencies.py`, extender `get_current_super_admin`
  para que reciba `request: Request` y, justo antes de retornar el usuario, guarde
  `request.state.super_admin_id = user.id` — cambio aditivo, sin alterar sus `raise
  HTTPException(401/403, ...)` existentes ni su valor de retorno. **Antes de modificarla,
  confirmar que ningún characterization test la invoca directamente sin pasar `request`** (la
  auditoría de `research.md` § 2/§ 5 no encontró ninguno; volver a verificar con `grep -rn
  get_current_super_admin app/characterization_tests` antes de aplicar el cambio). Depende de:
  T008.
- [X] T016 [US2] En la rama `except Exception` de
  `../pos-backend/app/core/error_middleware.py` (creada en T007), agregar la llamada a
  `sentry_sdk.capture_exception(exc)` con `request_id` (de `request.state.request_id`), `user_id`
  (de `request.state.super_admin_id` si existe, si no `None`), `module="super-admin"` y
  `operation` (método HTTP + plantilla de ruta) como tags/contexto de Sentry — **nunca** el cuerpo
  de la solicitud, cabeceras de autorización, ni tokens (FR-012). Las ramas `DomainError`/
  `HTTPException` (errores de negocio esperados) no deben llamar a `capture_exception`. Depende
  de: T007, T015.
- [X] T017 [P] [US2] En `../pos-backend/app/api/v1/super_admin/router.py`, dentro del bloque
  `except Exception:` ya existente en `create_tenant` (línea ~154, alrededor de
  `send_email_task.delay(...)`), agregar una llamada a `sentry_sdk.capture_exception()` **sin
  modificar el resto del bloque**: la respuesta sigue siendo `{"status": "ok", ...}` y el
  `logger.warning(...)` existente no se toca (`research.md` § 9, FR-007). Depende de: T009.

**Checkpoint**: User Story 2 es funcional y verificable de forma independiente — en producción,
las fallas técnicas inesperadas del módulo (incluida la del correo de bienvenida) quedan visibles
en Sentry con contexto; nada de eso ocurre fuera de producción ni para errores de negocio
esperados.

---

## Phase 5: User Story 3 - Continuidad para el panel de super-admin existente (Priority: P3)

**Goal**: Las pantallas actuales del panel de super-admin en `pos-heladeria` siguen mostrando el
mismo mensaje de error real que hoy, sin ningún cambio en ese repositorio.

**Independent Test**: Reproducir, contra el backend ya cambiado, los mismos escenarios de error
que hoy dispara el panel, y confirmar que el campo `detail` de nivel superior de cada respuesta
sigue trayendo el mismo texto que antes del cambio.

### Tests for User Story 3

- [X] T018 [US3] Extender
  `../pos-backend/app/characterization_tests/test_super_admin_error_envelope.py` (de T010) con
  una aserción explícita, en cada caso ya cubierto, de que `response.json()["detail"] ==
  response.json()["error"]["message"]` (campo de compatibilidad de nivel superior, `spec.md` §
  Clarifications, `data-model.md` § 2). Depende de: T010, T013.

### Implementation for User Story 3

- [X] T019 [P] [US3] Verificación manual/documental (sin cambios de código): releer
  `../pos-heladeria/src/app/modules/super-admin/services/tenant.service.ts`,
  `.../services/plan.service.ts`, `.../services/payment-method-catalog.service.ts` y
  `.../services/super-admin-users.service.ts`, confirmar que los cuatro siguen leyendo
  `err.error.detail`/`err.error.message` tal como hoy, y anotar en el PR/registro de la
  implementación que **no se modificó ningún archivo de `pos-heladeria`** para este spec
  (`research.md` § 4, SC-006). Depende de: T018.

**Checkpoint**: Las tres historias de usuario funcionan de forma independiente y verificable; el
panel de super-admin existente no sufre ninguna regresión visible.

---

## Phase 6: Polish & Cross-Cutting Concerns

**Purpose**: Cerrar la trazabilidad y dejar documentado el patrón para migrar otros módulos, tal
como pide el objetivo 8 del pedido original.

- [X] T020 [P] Documentar, como comentario al inicio de
  `../pos-backend/app/core/error_middleware.py`, cómo activar este mismo mecanismo para otro
  módulo (agregar su prefijo de ruta al registrar el middleware en `app/main.py`, sin duplicar
  `domain_errors.py`/`error_response.py`/`error_middleware.py`) — el patrón que
  `plan.md` § Project Structure promete dejar documentado.
- [X] T021 Ejecutar de punta a punta los 5 pasos de `quickstart.md` contra el entorno local
  (`../pos-backend`) y registrar el resultado de cada uno. **Parcial**: pasos 1-2 y 5 ejecutados
  y en verde (con un venv temporal creado para esta sesión — no versionado — instalando
  exactamente `requirements.txt`, dado que este entorno no traía uno). Paso 3 verificado solo en
  su mitad no dependiente de Postgres/Redis (`test_super_admin_sentry_integration.py` cubre el
  gate de entorno de `error_middleware.py`); la mitad que exige importar `app.main`
  (`sentry_sdk.Hub.current.client is None` contra la app real) no se ejecutó: `app.main` llama
  `initialize_database()`/`token_blocklist.ping()` desde su primera línea y este sandbox no tiene
  Postgres/Redis reales, ni acceso a Docker. Paso 4 (Sentry en "prod" contra `create_tenant`) no
  se ejecutó por el mismo motivo, agravado por `tenant_create()` (research.md § 11 /
  `test_tenant_plan_assignment.py`: exige Postgres real desde su primera línea, ya era así antes
  de este spec). Pendiente de confirmarse contra un entorno con esos servicios reales.
- [X] T022 [P] Ejecutar `python -m unittest discover -s app/characterization_tests` en
  `../pos-backend` (suite completa) y confirmar que ningún characterization test de **otro**
  módulo quedó en rojo por este cambio.

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Fase 1)**: sin dependencias — puede iniciar de inmediato.
- **Foundational (Fase 2)**: depende de que Setup esté completo — bloquea las tres historias de
  usuario.
- **User Stories (Fases 3-5)**: todas dependen de que Foundational esté completo.
  - US1 no depende de US2 ni de US3.
  - US2 depende de que exista el middleware de US1 (T007/T008), pero sus tests (T014) y su
    verificación son independientes de que T010-T013 estén terminadas.
  - US3 depende textualmente de la suite de tests que crea US1 (T010, extendida en T018) — es la
    única historia con una dependencia real sobre otra, porque solo verifica un campo adicional
    (`detail`) en los mismos casos que US1 ya construyó.
- **Polish (Fase 6)**: depende de que las historias que se vayan a entregar estén completas.

### Parallel Opportunities

- T001-T004 (Fase 1) son totalmente paralelas entre sí (archivos distintos).
- T011 puede correr en paralelo con T010 (archivos de test distintos) una vez completada la Fase
  2.
- T017 puede correr en paralelo con T015/T016 (archivo distinto: `router.py` vs.
  `dependencies.py`/`error_middleware.py`).
- T019 puede correr en paralelo con T018 (verificación en otro repositorio, sin dependencia de
  código).
- T020 y T022 son paralelas entre sí en la Fase 6.

---

## Parallel Example: Setup

```bash
# Lanzar juntas las cuatro tareas de la Fase 1:
Task: "Agregar SENTRY_DSN a Settings en app/core/config.py"
Task: "Crear app/core/domain_errors.py"
Task: "Crear app/core/error_response.py"
Task: "Crear app/core/error_middleware.py"
```

## Parallel Example: User Story 1

```bash
# Una vez completada la Fase 2, lanzar juntas:
Task: "Tests de envelope de error en test_super_admin_error_envelope.py (T010)"
Task: "Test de control fuera del módulo en test_error_middleware_scope.py (T011)"
```

---

## Implementation Strategy

### MVP First (User Story 1 únicamente)

1. Completar Fase 1: Setup.
2. Completar Fase 2: Foundational (crítico — bloquea las tres historias).
3. Completar Fase 3: User Story 1.
4. **Detenerse y validar**: correr T010-T012 y confirmar que pasan de forma independiente.
5. Este punto ya es un incremento entregable: todo el módulo super-admin responde con el
   envelope consistente y seguro, sin ninguna fuga de información técnica — aunque todavía sin
   observabilidad en Sentry ni verificación explícita de compatibilidad con el frontend.

### Incremental Delivery

1. Setup + Foundational → base lista.
2. + User Story 1 → validar de forma independiente → **MVP**: envelope consistente y seguro en
   todo el módulo.
3. + User Story 2 → validar de forma independiente → visibilidad de fallas inesperadas en
   producción, sin ruido.
4. + User Story 3 → validar de forma independiente → confirmación explícita de que el panel de
   super-admin existente no se rompió.
5. + Polish → patrón documentado para migrar otros módulos más adelante.

---

## Notes

- Ninguna tarea de este documento modifica `app/core/crud.py`, `app/core/plan_limits.py`, ni
  ninguna función de `app/core/dependencies.py` distinta de `get_current_super_admin`
  (`plan.md` § Constraints, `research.md` § 2) — es la restricción central que hace posible que
  las historias sean independientes sin arriesgar a otros módulos ni a los characterization tests
  ya existentes de super-admin.
- T015 es la única tarea que cambia la firma de una función compartida con otros endpoints
  (`get_current_super_admin`); por eso lleva su propio paso de verificación explícito antes de
  aplicarse.
- Cometer (`git commit`) después de cada tarea o grupo lógico, como de costumbre en este
  proyecto.

## Estado de ejecución (`/speckit-implement`, 2026-09-02)

- **Todas las tareas T001-T022 implementadas y verificadas**, salvo lo anotado como parcial en
  T021. Corrección de diseño encontrada al escribir el código real (dos ajustes, ninguno cambia
  las decisiones de `research.md` 1-10): documentada en `research.md` § 11 — el mecanismo de
  traducción a HTTP terminó siendo un middleware de solo-`request_id`/excepciones-no-anticipadas
  más tres `@app.exception_handler` con guardia de prefijo, no un único `BaseHTTPMiddleware` que
  reescribe cualquier respuesta; y `TestClient` resultó ser un patrón nuevo para este repo, no uno
  ya usado (`plan.md`/`tasks.md` T010 corregidos).
- Suite completa ejecutada: `python -m unittest discover -s app/characterization_tests` →
  **610 tests, 609 en verde**. El único que falla
  (`test_tenant_plan_assignment.py`, `ModuleNotFoundError: No module named 'app.api.v1.admin'`)
  es **preexistente y no relacionado con este spec**: `app/main.py` perdió el import/montaje de
  `admin_router` por un cambio externo a esta sesión (detectado al re-leer el archivo, ver
  aviso del sistema), no por ninguna tarea de aquí — `test_tenant_plan_assignment.py` importa
  `app.api.v1.admin.schema`, un paquete que ya no existe en disco. No se tocó (Principio V: no
  arreglar algo fuera del alcance del spec sin que sea estrictamente necesario) — documentado
  para que el equipo lo revise.
- Entorno de ejecución: este sandbox no traía un virtualenv para `pos-backend` ni acceso a
  Docker/Postgres/Redis. Se creó un venv temporal (`/tmp/pos-backend-venv`, no versionado,
  descartable) instalando exactamente `requirements.txt`, sobre Python 3.14 (el único intérprete
  disponible aquí — distinto del `python:3.12-slim` del `Dockerfile`, aunque todas las
  dependencias fijadas instalaron y corrieron sin fricción). Recomendado: repetir la suite
  también contra 3.12 antes de dar el spec por cerrado, y correr el resto de `quickstart.md`
  contra un entorno con Postgres/Redis reales (pasos 3 completo y 4).

## Corrección posterior a T022 (misma sesión, contra el servidor real del usuario)

Al usar el feature en vivo (`POST /api/v1/super-admin/tenants` contra Postgres/Redis reales), el
usuario encontró un hueco no cubierto por T007/FR-008: `tenant_create()`
(`../pos-backend/app/core/db.py::122-142`, código preexistente, no tocado por este spec) atrapa
cualquier excepción inesperada de la creación de tenant y la relanza como
`HTTPException(500, "Internal server error")` **antes** de que llegue a `RequestIdMiddleware`.
Ese camino pasaba por `register_error_handlers`'s handler de `HTTPException` (envelope correcto,
verificado en vivo contra el servidor real del usuario), pero **no** quedaba logueado con
`request_id` ni reportado a Sentry — el reporte solo estaba cableado en el camino de excepción no
envuelta de `RequestIdMiddleware`.

**Corrección aplicada**: `register_error_handlers`'s handler de `HTTPException`
(`app/core/error_middleware.py`) ahora también llama `_report_unexpected_exception` cuando
`exc.status_code >= 500`, sin importar si la excepción llegó cruda o ya envuelta — un 500 sigue
siendo una falla técnica en cualquiera de los dos casos. Sentry recibe la cadena completa
(`HTTPException` + su causa original, por el encadenamiento implícito de Python), así que la
excepción real que lo originó sigue siendo diagnosticable ahí aunque el mensaje que ve el cliente
se mantenga genérico y seguro.

Test nuevo: `test_httpexception_500_ya_envuelta_tambien_reporta_a_sentry` en
`test_super_admin_sentry_integration.py`. Suite completa re-ejecutada tras el cambio: **611
tests, 610 en verde** (mismo único fallo preexistente y no relacionado de
`test_tenant_plan_assignment.py`, sección anterior). Verificado además en vivo contra el
servidor real del usuario (`fastapi dev`, Postgres/Redis reales): el envelope de error se
confirmó correcto antes y después de este cambio; el reporte a Sentry no se pudo confirmar en
vivo porque ese entorno corre con `ENVIRONMENT=dev` (correctamente, no debe reportar ahí) — queda
cubierto por el test nuevo contra un entorno simulado en `"prod"`.

## Segunda corrección (mismo hilo): formato del log de terminal

El usuario pidió, además, que la terminal muestre el **mismo contexto** que se envía a Sentry
(`module`/`operation`/`request_id`/`user_id`), sin importar el entorno — antes
`_report_unexpected_exception` solo incluía `operation`/`request_id` en el mensaje de log;
`module` y `user_id` solo viajaban como tags/usuario de Sentry, invisibles en la terminal.

**Corrección aplicada**: `_report_unexpected_exception` (`app/core/error_middleware.py`) calcula
`user_id` una sola vez y lo usa tanto en el log (`logger.exception("Falla técnica inesperada |
module=%s operation=%s request_id=%s user_id=%s", ...)`, incondicional — ya corría fuera del gate
de `ENVIRONMENT`, así que ya se veía en dev) como en el `scope.set_user` de Sentry. El traceback
completo se sigue adjuntando solo (`logger.exception` agrega `exc_info` después del mensaje).

Test nuevo: `test_terminal_muestra_el_mismo_contexto_que_se_envia_a_sentry_en_cualquier_entorno`
(usa `assertLogs` para verificar los cuatro campos en la línea de log). Suite completa
re-ejecutada: **612 tests, 611 en verde** (mismo único fallo preexistente). Verificado en vivo
contra el servidor real del usuario una vez más (`POST /api/v1/super-admin/tenants`, sigue en
500 por la migración rota, sin relación con este spec).
