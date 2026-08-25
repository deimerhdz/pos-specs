# Quickstart: validar Planes de Suscripción por Tenant

Guía de ejecución para comprobar que la implementación cumple `spec.md`. No repite firmas ni
columnas ya detalladas en [data-model.md](./data-model.md) y `contracts/` — solo enlaza a ellas.

**Prerequisitos**: entorno virtual de `pos-backend` (Python 3.14), ejecutado desde la raíz de
`../pos-backend` (sibling de este repo `pos-specs`).

```bash
cd ../pos-backend
source env/bin/activate
```

No hace falta PostgreSQL real para los tests: `app/characterization_tests/fixtures.py` crea
SQLite en memoria. El test de concurrencia (US3, FR-015) usa dos sesiones/conexiones sobre la
misma base SQLite en memoria compartida, mismo patrón que cualquier test que ejercite
`with_for_update()` en este repo — si SQLite no soporta el lock de forma representativa, se
documenta explícitamente en el test y se complementa con una prueba manual contra Postgres real
(ver "Verificación de concurrencia contra Postgres" más abajo). Los tests de vencimiento (US5)
manipulan `plan_iniciado_en`/`plan_vence_en` directamente vía fixtures — no hace falta esperar
un mes/año real ni mockear el reloj del sistema completo, solo construir un `Tenant` cuya
`plan_vence_en` ya quedó en el pasado.

## Paso 0 — Confirmar la línea base antes de tocar código

```bash
python -m unittest discover -s app/characterization_tests -p 'test_*.py' -v
```

**Resultado esperado**: todo en verde. Ningún `"CONGELA comportamiento actual:"` se toca por esta
spec (research.md confirma que ninguno asume ausencia de límites, de gating de módulos, ni de
vencimiento).

## US1 — El Super Admin define planes con límites, accesos y precios

Fichero nuevo: `app/characterization_tests/test_super_admin_plans.py`, con `plan_fixtures.py`.

1. `POST /super-admin/plans` con `name="Básico"`, `mesas_limit=5`, `inventario_access=false`
   (autenticado como super admin) → `201`, el resto de límites en su default (`0`), accesos en
   `false` y precios en `null` sin haberlos enviado (Acceptance Scenario 1,
   [contracts/super-admin-plans.md](./contracts/super-admin-plans.md), FR-001/FR-002).
2. `GET /super-admin/plans` → el plan aparece listado (SC-001: "sin leer código").
3. `PATCH /super-admin/plans/{id}` con `mesas_limit=8` → `200`; un tenant de prueba con este plan
   asignado (ver US2) puede crear una sexta/séptima/octava mesa de inmediato, sin ninguna acción
   adicional (Acceptance Scenario 2, FR-014).
4. `PATCH /super-admin/plans/{id}` con `mesas_limit=null` → `200`; el mismo tenant puede crear
   cualquier cantidad de mesas sin bloqueo (Acceptance Scenario 3, FR-007).
5. `POST /super-admin/plans` con `name="Pro"`, `precio_mensual=89000.00`, `precio_anual=890000.00`
   → `201`, ambos precios guardados independientemente (Acceptance Scenario 4, FR-016).
6. Intentar los mismos endpoints sin `is_super_admin` (usuario `ADMIN` de tenant) → `403`.

```bash
python -m unittest app.characterization_tests.test_super_admin_plans -v
```

## US2 — El Super Admin asigna, cambia y renueva el plan de un tenant

Fichero nuevo: `app/characterization_tests/test_tenant_plan_assignment.py`.

1. `POST /admin/tenants` **sin** `plan_id` → `422`, tenant no creado (Acceptance Scenario 1, FR-004,
   edge case "no existe tenant sin plan, ni siquiera transitoriamente").
2. `POST /admin/tenants` **sin** `ciclo_facturacion` (pero con `plan_id`) → `422` (FR-017 exige
   elegirlo explícitamente, research.md Decisión 15).
3. `POST /admin/tenants` con `plan_id` de "Básico" y `ciclo_facturacion="mensual"` → `201`; el
   tenant creado tiene `plan_id` correcto, `plan_iniciado_en` = ahora, y `plan_vence_en` = ahora +
   1 mes, todo desde el primer `SELECT` (misma transacción, `data-model.md`) (Acceptance Scenario
   5, FR-004/FR-017/FR-018).
4. `PATCH /super-admin/tenants/{id}` con `plan_id` de "Pro" y `ciclo_facturacion="anual"` → `200`;
   el tenant queda regido por "Pro" de inmediato, con `plan_vence_en` recalculada a un año desde
   ese momento (Acceptance Scenario 2, FR-010).
5. `PATCH /super-admin/tenants/{id}` con `ciclo_facturacion="mensual"` sobre un plan cuyo
   `precio_mensual` es `null` → `409` (Acceptance Scenario 6, FR-017).
6. Con un tenant en un plan de `mesas_limit=5` y 8 mesas ya creadas (posible porque fueron creadas
   bajo un plan anterior sin ese límite, o porque el límite se bajó después vía US1.3),
   `PATCH /super-admin/tenants/{id}` no las borra ni las desactiva — `GET /orders/tables` sigue
   devolviendo las 8; una novena creación → `403` (Acceptance Scenario 3, FR-011).
7. Con un tenant en un plan con `inventario_access=true` y datos cargados en inventario,
   `PATCH /super-admin/tenants/{id}` a un plan con `inventario_access=false` → `GET
   /inventory/items` pasa de `200` a `403`; los datos siguen en la base (consulta directa a la
   tabla en el test) pero ningún endpoint del módulo responde (Acceptance Scenario 4, FR-012).
8. Con un tenant cuyo `plan_vence_en` quedó en el pasado (fixture, sin pasar por un `PATCH`
   real), `PATCH /super-admin/tenants/{id}` con el mismo `plan_id` y `ciclo_facturacion` de
   siempre → `200`, `plan_vence_en` se recalcula al futuro (Acceptance Scenario 7 — "renovar" no
   es más que este mismo endpoint, research.md Decisión 16).

```bash
python -m unittest app.characterization_tests.test_tenant_plan_assignment -v
```

## US3 — El sistema bloquea la creación de recursos que exceden el límite del plan

Fichero nuevo: `app/characterization_tests/test_plan_resource_limits.py`.

1. Tenant con `mesas_limit=5` y 5 mesas ya creadas → `POST /orders/tables` → `403` con mensaje que
   menciona "5"; `GET /orders/tables` sigue devolviendo exactamente 5 (Acceptance Scenario 1,
   [contracts/tenant-plan-enforcement.md](./contracts/tenant-plan-enforcement.md), FR-006).
2. Mismo tenant con solo 4 mesas → `POST /orders/tables` → `201`, sin ningún campo de aviso
   (Acceptance Scenario 2).
3. Tenant con `mesas_limit=null` → crear 10 mesas seguidas → todas `201` (Acceptance Scenario 3,
   FR-007).
4. Repetir el caso 1 para cajas, usuarios, productos y métodos de pago activos, confirmando el
   mismo mecanismo de bloqueo + mensaje para cada uno (Acceptance Scenario 4, data-model.md tabla
   de reglas de conteo — en particular, confirmar que desactivar un usuario/caja/producto **no**
   libera cupo, y que desactivar un método de pago **sí** libera cupo de
   `metodos_pago_activos_limit`).
5. **Concurrencia (FR-015)**: tenant con `mesas_limit=5` y 4 mesas ya creadas (un cupo libre);
   disparar dos `POST /orders/tables` desde dos sesiones/hilos distintos apuntando al mismo cupo
   → exactamente una recibe `201`, la otra recibe `403` con el mismo mensaje de límite alcanzado,
   sin importar el orden de llegada; `GET /orders/tables` termina con exactamente 5 mesas, nunca 6
   (edge case de `spec.md`, research.md Decisión 5).

```bash
python -m unittest app.characterization_tests.test_plan_resource_limits -v
```

### Verificación de concurrencia contra Postgres (complementaria, no automatizada en CI)

Si el test SQLite del punto 5 no ejercita `with_for_update()` de forma representativa (SQLite no
tiene el mismo comportamiento de locks a nivel de fila que Postgres), repetir manualmente contra
una base Postgres real de desarrollo:

```bash
# Terminal 1 y Terminal 2, casi simultáneas, mismo tenant con 1 cupo libre:
curl -X POST http://localhost:8000/orders/tables -H "x-tenant-host: ..." -d '{"number": 99}'
```

**Resultado esperado**: una `201`, la otra `403`; `SELECT count(*) FROM tenant_schema.dining_tables`
= exactamente el límite configurado.

## US4 — El sistema deniega el acceso a módulos que el plan no incluye

Fichero nuevo: `app/characterization_tests/test_plan_module_access.py`.

1. Tenant con `compras_access=false` (pero `inventario_access=true`) → `GET /inventory/items` →
   `200` (inventario sí incluido); `POST /inventory/purchases` → `403` con mensaje claro
   (Acceptance Scenario 1, research.md Decisión 6, FR-009).
2. Mismo tenant, `GET /inventory/items` → confirma que "inventario" y "compras" se gatean de forma
   independiente dentro del mismo router (Acceptance Scenario 2).
3. Tenant con `promociones_access=false` → cualquier endpoint de `promotions/router.py` → `403`.

```bash
python -m unittest app.characterization_tests.test_plan_module_access -v
```

## US5 — El sistema bloquea automáticamente a un tenant cuyo plan vence sin renovarse

Fichero nuevo: `app/characterization_tests/test_plan_expiration.py`.

1. Tenant cuyo `plan_vence_en` está en el pasado (fixture) → `POST /orders/tables` (con cupo
   disponible bajo el límite del plan) → `403` con mensaje que indica que el plan venció
   (Acceptance Scenario 1,
   [contracts/tenant-plan-enforcement.md](./contracts/tenant-plan-enforcement.md), FR-019).
2. Mismo tenant → `GET /inventory/items` (con `inventario_access=true` en su plan) → `403` con el
   mismo mensaje de vencimiento, no un `200` ni un mensaje distinto al de módulos (Acceptance
   Scenario 2, FR-019).
3. Mismo tenant → sus mesas/cajas/productos/datos de inventario ya creados siguen existiendo
   (consulta directa a la tabla en el test) — el bloqueo no borra nada (Acceptance Scenario 3).
4. Tenant con `plan_vence_en IS NULL` (plan de transición o "sin vencimiento" elegido
   explícitamente) → `POST /orders/tables` y `GET /inventory/items` → nunca `403` por vencimiento,
   sin importar la fecha del sistema (Acceptance Scenario 4, FR-021).
5. Sobre el tenant del caso 1 (ya vencido y bloqueado), `PATCH /super-admin/tenants/{id}` (US2.8,
   renovación) → el siguiente `POST /orders/tables`/`GET /inventory/items` vuelve a `201`/`200` de
   inmediato, sin que el Tenant Admin haga nada (Acceptance Scenario 5, FR-020).

```bash
python -m unittest app.characterization_tests.test_plan_expiration -v
```

## US6 — El Tenant Admin consulta su plan, su consumo y su vencimiento

Fichero nuevo: `app/characterization_tests/test_plan_summary.py`.

1. Tenant con `mesas_limit=5` y 4 mesas creadas → `GET /plan` → `resources.mesas = {"used": 4,
   "limit": 5}`, junto con el estado de cada acceso de módulo, en una sola respuesta (Acceptance
   Scenario 1, [contracts/tenant-plan-enforcement.md](./contracts/tenant-plan-enforcement.md),
   FR-013).
2. Tenant con `productos_limit=null` → `GET /plan` → `resources.productos.limit = null`
   (Acceptance Scenario 2 — el frontend traduce esto a "ilimitado").
3. Tenant con `ciclo_facturacion="mensual"` y `plan_vence_en` en el futuro → `GET /plan` →
   `plan_vence_en` presente y `vencido: false` (Acceptance Scenario 3, FR-013).
4. Tenant con `plan_vence_en IS NULL` → `GET /plan` → `plan_vence_en: null`, `ciclo_facturacion:
   null` (el frontend muestra "sin vencimiento", Acceptance Scenario 4, FR-021).
5. `GET /plan` autenticado como `CASHIER` del tenant (no solo `ADMIN`) → `200` (research.md
   Decisión 7 — necesario para que el guard de navegación funcione para cualquier rol).

```bash
python -m unittest app.characterization_tests.test_plan_summary -v
```

## Migración de datos existentes

```bash
alembic upgrade head
```

**Resultado esperado** (SC-006/SC-007): cero filas en `shared.tenants` con `plan_id IS NULL`; cada
tenant existente antes de la migración sigue sin ningún bloqueo de límite, de módulo, ni de
vencimiento inmediatamente después (consulta manual: `GET /plan` de un tenant preexistente
muestra `limit: null` en las cinco características, los tres módulos en `true`, y `plan_vence_en:
null`/`vencido: false`, hasta que el Super Admin lo baje activamente a un plan real con su propio
ciclo vía US2).

## Frontend (`pos-heladeria`)

```bash
cd ../pos-heladeria
npm test
```

Specs nuevos relevantes: `plan.service.spec.ts` (nuevo, módulo `super-admin`),
`plan-summary.service.spec.ts` (nuevo, módulo `plan`), `plan-module.guard.spec.ts` (nuevo, incluye
el caso `vencido: true`). Validación manual end-to-end (SC-001, no automatizable sin navegador
real):

1. Como Super Admin, crear el plan "Básico" en `/super-admin/plans` con `mesas_limit=5`, sin
   acceso a inventario, y `precio_mensual=89000` → asignarlo a un tenant de prueba en
   `/super-admin/tenants` eligiendo ciclo mensual → confirmar, usando la aplicación, que ese
   tenant no puede crear una sexta mesa ni ver el módulo de inventario en la navegación, en menos
   de 10 minutos (SC-001/SC-002/SC-003).
2. Como Tenant Admin de ese tenant, abrir `dashboard/mi-plan` → confirmar que ve "4 de 5" para
   mesas, el estado de cada módulo, y la fecha de vencimiento (aproximadamente un mes desde la
   asignación), sin ayuda de soporte (SC-004).
3. Como Super Admin, subir `mesas_limit` a 8 en el mismo plan → confirmar, sin recargar sesión del
   Tenant Admin, que el siguiente intento de crear mesa ya no se bloquea (SC-005).
4. Como Super Admin, editar manualmente en la base de datos (o vía un tenant de prueba dedicado) la
   `plan_vence_en` de un tenant a una fecha pasada → confirmar que el Tenant Admin ve el módulo de
   inventario desaparecer de la navegación y cualquier intento de crear un recurso rechazado con
   el mensaje de vencimiento (SC-007) → renovar desde `/super-admin/tenants` → confirmar que el
   acceso se restaura de inmediato, sin que el Tenant Admin cierre sesión.
