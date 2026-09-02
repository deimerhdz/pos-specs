# Quickstart de validación: corrección de la referencia entre schemas al crear un tenant

Guía para comprobar que el hotfix del feature 070 resuelve el error reportado, sin romper nada
más. Comandos desde `../pos-backend`, contra un entorno con PostgreSQL real (local o el de
producción) — este defecto no es reproducible contra SQLite en memoria (research.md § 5).

**Prerrequisito**: spec 069 (corrección del `NameError` de Alembic) ya debe estar aplicada — de
lo contrario, la creación de un tenant vuelve a fallar antes, por esa otra causa.

## 1. Crear un tenant con usuario, de punta a punta

Usando el endpoint real (`POST /api/v1/super-admin/tenants`, ver
`specs/068-manejo-excepciones-super-admin/quickstart.md` si hace falta un token de super-admin):

```bash
curl -s -i -X POST http://localhost:8000/api/v1/super-admin/tenants \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "tenant_name": "Verificación 070",
    "schema_name": "verificacion_070",
    "host": "verificacion-070",
    "name": "Admin de prueba",
    "email": "admin-verificacion-070@example.com",
    "plan_id": "<uuid de un plan existente>",
    "ciclo_facturacion": null
  }'
```

**Antes del fix**: `500`, con el envelope de error de spec 068
(`error.code = "HTTP_500"`), `detail: "Internal server error"` — el `NoReferencedTableError` real
solo visible en el log del servidor (o en Sentry, si `ENVIRONMENT=prod`, por el hotfix de
logging de spec 068).

**Se espera después del fix**: `200`, con `{"status": "ok", "tenant": "Verificación 070"}`.
Confirmar además que el schema del tenant nuevo tiene **todas** sus tablas, incluida
`payment_methods`:

```bash
psql "$DATABASE_URL" -c "\dt verificacion_070.*" | grep payment_methods
```

Debe listarla.

## 2. El tenant nuevo puede usar el catálogo compartido de métodos de pago

Confirma que la relación no solo "no rompió la creación", sino que quedó bien formada
(spec.md Acceptance Scenario 2): con el super-admin, activar en el tenant recién creado un
método de pago del catálogo de la plataforma (`POST` sobre el endpoint de métodos de pago del
tenant, referenciando un `catalog_id` del catálogo compartido) y confirmar que la operación
funciona con normalidad, sin ningún error relacionado con la FK.

## 3. Ningún otro tenant ni el catálogo compartido cambiaron

```bash
psql "$DATABASE_URL" -c "SELECT count(*) FROM shared.payment_method_catalog;"
```

Debe devolver el mismo número de filas que tenía antes de aplicar el fix — confirma que la
corrección no duplicó ni recreó la tabla compartida (research.md § 3).

## 4. Suite de regresión

```bash
python -m unittest discover -s app/characterization_tests
```

Se espera que siga en verde — ningún characterization test cubre esta función (research.md § 5),
pero se corre igual como red de seguridad.

## Criterio de aceptación de esta guía

El hotfix se considera validado cuando los pasos 1, 2 y 3 pasan en local. Recomendado repetir el
paso 1 contra producción antes de cerrar el incidente (mismo criterio que spec 069).
