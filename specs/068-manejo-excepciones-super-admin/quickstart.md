# Quickstart de validación: manejo de excepciones en super-admin

Guía para comprobar, de punta a punta, que el feature 068 quedó implementado según `spec.md` y
`contracts/error-envelope.md`. Todos los comandos corren desde `../pos-backend`.

## Prerrequisitos

- Entorno local del backend levantado (`docker-compose.yml`: Postgres + Redis) con
  `ENVIRONMENT=dev` en `.env` (valor por defecto del repo).
- Un usuario super-admin sembrado (`app/scripts/seed_super_admin.py`, ya existente) y su token de
  acceso.

## 1. Los characterization tests existentes siguen en verde (no deben tocarse)

```bash
python -m unittest app.characterization_tests.test_super_admin_plans -v
python -m unittest app.characterization_tests.test_super_admin_payment_catalog -v
```

**Resultado esperado**: ambos suites pasan exactamente igual que antes del cambio (`research.md`
§ 5 explica por qué el middleware nuevo no los afecta: invocan las funciones del router
directamente, sin pasar por la app ASGI).

## 2. El envelope de error aparece en las respuestas reales del módulo (vía `TestClient`)

```bash
python -m unittest app.characterization_tests.test_super_admin_error_envelope -v
```

Escenarios que ese archivo nuevo debe cubrir (uno por fila de la tabla de `contracts/error-envelope.md`):

| Escenario | Cómo provocarlo | Se espera |
|---|---|---|
| Tenant inexistente | `PATCH /api/v1/super-admin/tenants/{uuid-inventado}` | 404, `error.code == "NOT_FOUND"` (o `"TENANT_NOT_FOUND"`), `detail` presente |
| Plan inexistente | `PATCH /api/v1/super-admin/tenants/{id}` con `plan_id` inventado | 404 |
| Ciclo sin precio definido | `PATCH .../tenants/{id}` con un plan sin precio para el ciclo elegido | 409, `error.code == "CONFLICT"` |
| Nombre de plan duplicado | `POST /api/v1/super-admin/plans` con un `name` ya existente | 409 |
| Sin rol de super-admin | Cualquier endpoint con un token de usuario de tenant normal | 403, sin revelar si el recurso existe |
| Sin token / token inválido | Cualquier endpoint sin `Authorization` | 401 |
| Falla técnica inesperada | Forzar una excepción no controlada (p. ej. monkeypatchear la sesión de DB para que lance) | 500, `error.code == "INTERNAL_ERROR"`, **sin** traza ni mensaje interno en el cuerpo |
| Ruta de control fuera del módulo | Un error conocido en otro router (p. ej. `GET /api/v1/products/{uuid-inventado}`) | Sigue devolviendo `{"detail": "..."}` plano, **sin** los campos `success`/`error`/`request_id` — confirma que el middleware no se activó fuera de super-admin |

## 3. Sentry no se activa fuera de producción

```bash
ENVIRONMENT=dev python -c "
from app.main import app
import sentry_sdk
assert sentry_sdk.Hub.current.client is None, 'Sentry no debe inicializarse fuera de prod'
print('OK: Sentry inactivo en dev')
"
```

Repetir forzando una de las fallas de la tabla anterior con `ENVIRONMENT=dev`: la respuesta al
llamador debe ser idéntica a la de producción (mismo envelope), y no debe registrarse ningún
evento en Sentry (verificable interceptando `sentry_sdk.capture_exception` en el test, o
simplemente por la aserción de arriba: sin cliente inicializado, la llamada es un no-op).

## 4. Sentry se activa en producción, solo para fallas inesperadas, con el contexto esperado

Con `ENVIRONMENT=prod` y un `SENTRY_DSN` de prueba (o mockeando `sentry_sdk.capture_exception`):

- Provocar el escenario "Falla técnica inesperada" de la tabla del paso 2 → debe haber
  exactamente una llamada a `sentry_sdk.capture_exception`, con `request_id`, `user_id` del
  super-admin, `module="super-admin"` y `operation` en el contexto/tags del evento (FR-008,
  `data-model.md` § 3).
- Provocar cualquiera de los escenarios de error de negocio esperado (404/409/403/401 de la tabla
  del paso 2) → **cero** llamadas a `sentry_sdk.capture_exception` (FR-009).
- Provocar la falla del correo de bienvenida en `create_tenant` (mockear el envío para que lance)
  → la respuesta sigue siendo éxito (`{"status": "ok", ...}`, sin envelope de error, FR-007), y sí
  hay una llamada a `sentry_sdk.capture_exception` (visibilidad técnica sin afectar la respuesta,
  `research.md` § 9).

## 5. El panel de super-admin existente no se rompe

No requiere levantar `pos-heladeria`: basta con confirmar en el paso 2 que cada respuesta de
error trae el campo `detail` de nivel superior con el mismo texto que antes del cambio. Los 4
servicios de `pos-heladeria/src/app/modules/super-admin/` siguen leyendo ese campo sin
modificación (SC-006) — verificación manual opcional: abrir el panel contra este backend y
provocar un error de negocio conocido (p. ej. asignar un plan inexistente) y confirmar que el
mensaje mostrado es el mismo que antes del cambio.

## Criterio de aceptación de esta guía

El feature se considera validado cuando los pasos 1 a 4 pasan en CI/local y el paso 5 se
confirma al menos una vez manualmente. No se requiere ningún cambio en `pos-heladeria` para
completar este spec.
