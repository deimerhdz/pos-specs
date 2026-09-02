# Data Model: Manejo de excepciones y respuestas de error consistentes en el módulo super-admin

Este spec no introduce ni modifica ninguna tabla, columna ni migración (Constitution Check §
Principio VIII: N/A). Las "entidades" de este feature son estructuras en memoria/tipos, no datos
persistidos.

## 1. Jerarquía de excepciones de dominio (`app/core/domain_errors.py`, nuevo)

Sin ningún import de FastAPI/Starlette — son excepciones de Python puras, verificables sin
levantar una solicitud HTTP (FR-006).

```text
DomainError (base, abstracta)
├── attrs: code: str, message: str, details: dict | None = None
│
├── EntityNotFoundError(entity: str, identifier: Any)
│     code por defecto: f"{ENTITY}_NOT_FOUND" (p. ej. "TENANT_NOT_FOUND")
│     → HTTP 404
│
├── ConflictError
│     → HTTP 409
│
├── InvalidStateError
│     → HTTP 409
│     (transición/operación no válida para el estado actual del recurso)
│
├── BusinessRuleViolation
│     → HTTP 400
│     (regla de negocio incumplida que no encaja en las anteriores)
│
├── UnauthorizedError
│     → HTTP 401
│     (identidad ausente/ inválida — distinto de "sin permiso")
│
└── ForbiddenError
      → HTTP 403
      (identidad válida, sin el rol/permiso requerido — p. ej. no ser super-admin)
```

No se define `TenantAccessDenied` como clase propia: por el hallazgo de `research.md` § 3, este
módulo no tiene el concepto de "tenant intentando acceder a datos de otro tenant" (el actor es un
super-admin global, no un usuario de tenant) — el caso que sí existe ("no es super-admin") ya lo
cubre `ForbiddenError`, y es exactamente lo que `get_current_super_admin` produce hoy vía
`HTTPException(403, ...)`, sin necesidad de tocar esa dependencia.

Estas clases son **infraestructura disponible**, no de uso obligatorio en cada `raise`: por la
decisión de `research.md` § 2/§ 8, el código existente de super-admin sigue delegando en
`get_or_404`/`ensure_unique`/`validate_billing_cycle_price` (que siguen lanzando `HTTPException`,
sin cambios). El middleware (sección 3) traduce ambos orígenes al mismo envelope.

## 2. Respuesta de error estándar (envelope)

Estructura de la respuesta JSON para toda solicitud fallida al módulo super-admin. Construida en
`app/core/error_response.py` (nuevo).

| Campo | Tipo | Descripción |
|---|---|---|
| `success` | `bool` | Siempre `false` en una respuesta de error. |
| `error.code` | `str` | Código estable en mayúsculas/snake (p. ej. `NOT_FOUND`, `TENANT_NOT_FOUND`, `CONFLICT`, `FORBIDDEN`, `INTERNAL_ERROR`). Ver mapeo en `research.md` § 8. |
| `error.message` | `str` | Mensaje legible, seguro para mostrar a quien llama. Nunca contiene SQL, trazas, rutas de archivo, credenciales ni tokens (FR-003). |
| `error.details` | `object \| null` | Información estructurada adicional opcional (p. ej. qué campo entró en conflicto). `null` cuando no aplica. |
| `request_id` | `str` (UUID) | Generado por el middleware al inicio de cada solicitud a super-admin; permite correlacionar con logs/Sentry (FR-015, SC-003). |
| `detail` | `str` | **Campo de compatibilidad** (spec.md § Clarifications): igual a `error.message`. Permite que los 4 servicios existentes de `pos-heladeria` (que leen `err.error.detail`) sigan funcionando sin cambios. |

Ejemplo (404):

```json
{
  "success": false,
  "error": {
    "code": "TENANT_NOT_FOUND",
    "message": "No existe ese tenant",
    "details": null
  },
  "request_id": "8f14e45f-ceea-4c9d-b7c1-3f2a1e1c9a3b",
  "detail": "No existe ese tenant"
}
```

Ejemplo (409, sin código específico por entidad — camino que hoy pasa por `ensure_unique`):

```json
{
  "success": false,
  "error": {
    "code": "CONFLICT",
    "message": "Ya existe un plan con ese nombre",
    "details": null
  },
  "request_id": "1c2b3a4d-5e6f-4789-9abc-0def12345678",
  "detail": "Ya existe un plan con ese nombre"
}
```

Ejemplo (500, falla técnica inesperada — nunca expone la causa real):

```json
{
  "success": false,
  "error": {
    "code": "INTERNAL_ERROR",
    "message": "Ocurrió un error inesperado. Intenta de nuevo.",
    "details": null
  },
  "request_id": "aabbccdd-1122-3344-5566-77889900aabb",
  "detail": "Ocurrió un error inesperado. Intenta de nuevo."
}
```

## 3. Evento de monitoreo de errores (Sentry)

No es una entidad persistida por esta aplicación — es el payload que `sentry_sdk.capture_exception`
envía a la plataforma externa, solo cuando `ENVIRONMENT == "prod"` (`research.md` § 6) y solo para
`Exception` no clasificada como `DomainError`/`HTTPException` de negocio esperada (FR-008/FR-009).

| Campo de contexto | Origen |
|---|---|
| `request_id` | El mismo UUID generado por el middleware para esa solicitud (`request.state.request_id`) |
| `tenant_id` | `None` para este módulo (actor global) — se omite/`null` explícitamente; ver `research.md` § 3 |
| `user_id` | `id` del super-admin autenticado (`request.state`/`get_current_super_admin`), si la solicitud llegó a autenticarse |
| `module` | Constante `"super-admin"` |
| `operation` | Método + ruta (p. ej. `"PATCH /super-admin/tenants/{tenant_id}"`) |

Explícitamente **no** incluido en el payload (FR-012): cuerpo completo de la solicitud, cabeceras
de autorización, contraseñas, tokens.

## 4. Configuración nueva (`app/core/config.py`, `Settings`)

| Campo | Tipo | Default | Nota |
|---|---|---|---|
| `SENTRY_DSN` | `Optional[str]` | `None` | Si es `None`, Sentry nunca se inicializa, sin importar el entorno (defensa adicional a `research.md` § 6). |

No se agrega ningún campo nuevo de "entorno" (se reutiliza `ENVIRONMENT` ya existente, decisión
de `research.md` § 6).
