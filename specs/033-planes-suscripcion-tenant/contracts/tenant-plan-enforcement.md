# Contrato: Cumplimiento del Plan (límites, accesos, vencimiento, consumo del tenant)

Cubre **Historias de Usuario 3, 4, 5 y 6**. No introduce un router monolítico nuevo: extiende
cinco endpoints de creación ya existentes (helper compartido `app/core/plan_limits.py`), dos
routers existentes (gating de módulo) y agrega un único endpoint de lectura nuevo (`GET /plan`).
El bloqueo por vencimiento (Historia de Usuario 5) no agrega ningún endpoint propio — se resuelve
dentro de la misma verificación que ya usan las Historias 3 y 4 (research.md Decisión 14), así que
se documenta como una sección de este mismo contrato, no uno separado.

## Bloqueo por límite numérico (Historia de Usuario 3)

Aplica a los cinco endpoints de creación ya existentes, sin cambiar su forma de request/response
— solo agregan la posibilidad de una respuesta `403` nueva antes del `201` habitual:

| Endpoint existente | Recurso | Clave de `Plan` |
|---|---|---|
| `POST /orders/tables` | Mesa | `mesas_limit` |
| `POST /cash/registers` | Caja | `cajas_limit` |
| `POST /users` | Usuario | `usuarios_limit` |
| `POST /products` | Producto | `productos_limit` |
| `POST /sales/payment-methods` (o `PATCH .../{id}` con `active: true`, según cómo termine
  resolviéndose la activación en la implementación de esta spec y de la 032 en paralelo) | Método de pago activo | `metodos_pago_activos_limit` |

**Response 403** (nueva, en cualquiera de los cinco, cuando el límite ya está alcanzado):

```json
{ "detail": "Límite de mesas alcanzado (5). Actualiza tu plan para crear más." }
```

El mensaje sigue el mismo patrón para los cinco recursos, sustituyendo el nombre del recurso y el
número (FR-006, Acceptance Scenario 1 de Historia de Usuario 3): `"Límite de {recurso} alcanzado
({N}). Actualiza tu plan para crear más."` No se crea el registro (rollback completo de la
solicitud) — verificado por `quickstart.md`.

**Sin bloqueo** cuando el plan tiene esa característica en `NULL` (ilimitado, Acceptance Scenario
3) o cuando el conteo actual es menor al límite (Acceptance Scenario 2) — en ambos casos la
respuesta es exactamente la misma que hoy (`201`, sin ningún campo nuevo).

**Concurrencia (FR-015)**: dos solicitudes casi simultáneas de creación del mismo recurso para el
mismo tenant, con exactamente un cupo disponible, nunca ambas reciben `201` — como máximo una
tiene éxito, la otra recibe el mismo `403` de arriba (ver data-model.md, diagrama de bloqueo por
límite; verificado con un test dedicado en `quickstart.md`).

## Bloqueo por acceso a módulo (Historia de Usuario 4)

Aplica a **todos** los endpoints de `app/api/v1/inventory/router.py` (clave `inventario_access`,
salvo los cuatro de compras) y `app/api/v1/promotions/router.py` (clave `promociones_access`,
router completo). Los cuatro endpoints de compras (`GET/POST /inventory/purchases`,
`POST /inventory/purchases/order`, `POST /inventory/purchases/{id}/receive`) requieren además
`compras_access`.

**Response 403** (nueva, cuando el módulo no está incluido en el plan vigente):

```json
{ "detail": "Tu plan actual no incluye el módulo de inventario." }
```

Mensaje análogo para `"...módulo de compras."` / `"...módulo de promociones."` (FR-009,
Acceptance Scenario 1 de Historia de Usuario 4). Se aplica a **todo** endpoint del módulo —
incluyendo los de solo lectura (`GET`) — no solo a los de creación: el módulo queda completamente
inaccesible, no en modo de solo lectura (edge case de `spec.md`).

**Sin bloqueo** cuando `Plan.<módulo>_access = true` — comportamiento idéntico al actual, sin
ningún campo nuevo en la respuesta (Acceptance Scenario 2 de Historia de Usuario 4).

## Bloqueo por vencimiento (Historia de Usuario 5)

Aplica a **exactamente los mismos puntos de entrada** que las dos secciones anteriores — los cinco
endpoints de creación y los routers de inventario/compras/promociones — porque
`ensure_plan_not_expired(tenant)` corre como primer paso dentro de `enforce_plan_limit` y
`require_module_access` (research.md Decisión 14, data-model.md diagrama "Bloqueo por
vencimiento"). No hay una tabla de endpoints nueva que documentar: es la misma tabla de arriba.

**Response 403** (nueva, cuando `Tenant.plan_vence_en` ya pasó), en cualquiera de los puntos de
entrada de arriba, **antes** de evaluar el límite o el módulo específico:

```json
{ "detail": "Tu plan venció. Debe renovarse para seguir usando el sistema." }
```

Mismo mensaje sin importar si la solicitud original era de creación de un recurso o de acceso a un
módulo (FR-019, Acceptance Scenarios 1 y 2 de Historia de Usuario 5) — el vencimiento es un
bloqueo total, no específico de un recurso o módulo. Ningún dato existente cambia (Acceptance
Scenario 3).

**Sin bloqueo** cuando `Tenant.plan_vence_en IS NULL` (sin vencimiento, FR-021, Acceptance
Scenario 4) o cuando `Tenant.plan_vence_en` todavía no pasó — en ambos casos la validación
continúa normalmente con el límite/módulo específico, exactamente como en las dos secciones
anteriores.

**Renovación levanta el bloqueo de inmediato** (Acceptance Scenario 5): tras un `PATCH
/super-admin/tenants/{id}` exitoso (ver
[contracts/super-admin-plans.md](./super-admin-plans.md)), el siguiente intento del tenant ya no
recibe este `403` — no hay ninguna caché ni token que invalidar, la próxima solicitud simplemente
vuelve a leer `Tenant.plan_vence_en` ya actualizado.

## `GET /plan` (nuevo — consumo del tenant, Historia de Usuario 6)

Router nuevo `app/api/v1/plan/router.py`, prefix `/plan`. Accesible a cualquier usuario
autenticado del tenant (`get_current_user`, no `require_tenant_admin` — ver research.md Decisión
7 para la justificación de por qué no se restringe solo a ADMIN).

**Response 200** — `PlanSummaryResponse`:

```json
{
  "plan_name": "Básico",
  "ciclo_facturacion": "mensual",
  "plan_vence_en": "2026-09-24T00:00:00Z",
  "vencido": false,
  "resources": {
    "mesas": { "used": 4, "limit": 5 },
    "cajas": { "used": 1, "limit": 1 },
    "usuarios": { "used": 3, "limit": 3 },
    "productos": { "used": 40, "limit": null },
    "metodos_pago_activos": { "used": 1, "limit": 2 }
  },
  "modules": {
    "inventario": false,
    "compras": false,
    "promociones": true
  }
}
```

`limit: null` significa ilimitado (Acceptance Scenario 2 de Historia de Usuario 6 — el frontend
muestra "ilimitado" en vez de un número). `plan_vence_en: null` (junto con `ciclo_facturacion:
null`) significa "sin vencimiento" — el frontend muestra un indicador de "sin vencimiento" en vez
de una fecha (Acceptance Scenario 4 de Historia de Usuario 6, FR-021). `vencido` es un booleano
calculado (`plan_vence_en is not None and plan_vence_en < ahora`) que el frontend usa
directamente, sin tener que repetir la comparación de fechas él mismo. `used` se calcula con las
mismas reglas de conteo que `enforce_plan_limit` (data-model.md: todas las filas para
mesas/cajas/usuarios/productos, solo activas para métodos de pago) — sin lock, es una lectura de
solo consulta, no participa en la garantía de concurrencia de FR-015.

Esta misma respuesta es la que el frontend usa para decidir, antes de navegar, si oculta/bloquea
las rutas de inventario/promociones y la pestaña de compras (`plan-module.guard.ts`,
`inventory-page.component.ts`) — ahora también cuando `vencido: true`, sin importar lo que digan
los `modules.*` individuales, evitando que un usuario navegue a una pantalla que de todas formas
va a responder `403` en cada llamada (SC-004).

**Errores**: ninguno específico de esta spec — los mismos `401` de autenticación que cualquier
endpoint tenant-scoped.
