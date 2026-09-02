# Quickstart de validación: corrección de creación de tenant

Guía para comprobar que el hotfix del feature 069 resuelve el error reportado, sin romper nada
más. Comandos desde `../pos-backend`, contra un entorno con PostgreSQL real (local o el de
producción, según corresponda) — este defecto no es reproducible contra SQLite en memoria
(research.md § 5: no hay characterization tests que lo cubran; se verifica de punta a punta).

## 1. El catálogo completo de migraciones carga sin error

```bash
alembic history
```

**Antes del fix**: falla con `NameError: Can't invoke function 'f', as the proxy object has not
yet been established for the Alembic 'Operations' class.` (`args: ["ck__promotions__ck_promotion_type"]`
o cualquiera de las otras tres).

**Después del fix**: lista el historial completo de revisiones sin ningún error.

## 2. `alembic upgrade head` sigue aplicando exactamente lo mismo que antes

```bash
alembic upgrade head
```

**Se espera**: si la base de datos ya estaba al día antes del fix (las migraciones ya se habían
aplicado por otra vía, o el defecto solo afectaba a la *carga* del catálogo, no a su aplicación
ya consumada), este comando no reporta ningún cambio pendiente — confirma que el DDL que ambas
migraciones aplican no cambió (`research.md` § 4, `spec.md` FR-004).

## 3. Crear un tenant con usuario, de punta a punta

Usando el endpoint real (`POST /api/v1/super-admin/tenants`, requiere un token de super-admin
válido — ver `specs/068-manejo-excepciones-super-admin/quickstart.md` si hace falta generar
uno):

```bash
curl -s -i -X POST http://localhost:8000/api/v1/super-admin/tenants \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "tenant_name": "Verificación 069",
    "schema_name": "verificacion_069",
    "host": "verificacion-069",
    "name": "Admin de prueba",
    "email": "admin-verificacion-069@example.com",
    "plan_id": "<uuid de un plan existente>",
    "ciclo_facturacion": null
  }'
```

**Antes del fix**: `500`, con el envelope de error de spec 068 (`error.code = "HTTP_500"`),
`detail: "Internal server error"` — el `NameError` real solo visible en el log del servidor
(o, tras el hotfix de logging de spec 068, también en Sentry si `ENVIRONMENT=prod`).

**Se espera después del fix**: `200`, con `{"status": "ok", "tenant": "Verificación 069"}`
(forma actual de `create_tenant`, `app/api/v1/super_admin/router.py`). Verificar además:

```bash
psql "$DATABASE_URL" -c "SELECT id, name, host, schema FROM shared.tenants WHERE host = 'verificacion-069';"
psql "$DATABASE_URL" -c "\dn verificacion_069"   -- el schema del tenant existe
```

Ambas consultas deben devolver una fila / el schema debe existir.

## 4. Repetir el chequeo de otro módulo no relacionado

Confirmar que ninguna otra ruta cambió de comportamiento — por ejemplo, un endpoint cualquiera
de otro módulo que ya funcionaba sigue respondiendo igual. No se espera ningún cambio; este paso
es solo para descartar un efecto colateral inesperado.

## Criterio de aceptación de esta guía

El hotfix se considera validado cuando los pasos 1 y 3 pasan (el paso 2 es informativo, confirma
que no hay DDL pendiente inesperado). Recomendado ejecutar primero en local y, una vez
confirmado, repetir el paso 1 y 3 contra producción antes de cerrar el incidente.
