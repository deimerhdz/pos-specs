# Quickstart: validar Catálogo de Métodos de Pago Administrado por el Super Admin

Guía de ejecución para comprobar que la implementación cumple `spec.md`. No repite firmas ni
columnas ya detalladas en [data-model.md](./data-model.md) y `contracts/` — solo enlaza a ellas.

**Prerequisitos**: entorno virtual de `pos-backend` (Python 3.14), ejecutado desde la raíz de
`../pos-backend` (sibling de este repo `pos-specs`).

```bash
cd ../pos-backend
source env/bin/activate
```

No hace falta PostgreSQL ni Cloudflare R2 reales: `app/characterization_tests/fixtures.py` crea
SQLite en memoria; el presign a R2 para la imagen QR se mockea con `unittest.mock.patch`, mismo
patrón que usan los tests de `products`/`uploads` ya existentes.

## Paso 0 — Confirmar la línea base antes de tocar código

```bash
python -m unittest discover -s app/characterization_tests -p 'test_*.py' -v
```

**Resultado esperado**: todo en verde. Ningún `"CONGELA comportamiento actual:"` se toca por esta
spec (research.md confirma que no existe ninguno específico de `payment_methods`); solo se edita
directamente `test_sales_payment_methods.py`, que no lleva ese prefijo.

## US1 — El Super Admin administra el catálogo de la plataforma

Fichero nuevo: `app/characterization_tests/test_super_admin_payment_catalog.py`, con
`payment_catalog_fixtures.py`.

1. `POST /super-admin/payment-methods-catalog` con `name="Daviplata"`, `type="transfer"`,
   `fields=[{"key":"celular","required":true,"format":"numeric","length":10}]` (autenticado como
   super admin, `get_current_super_admin`) → `201`, `active=true` (Acceptance Scenario 1,
   [contracts/super-admin-catalog.md](./contracts/super-admin-catalog.md)).
2. `GET /super-admin/payment-methods-catalog` → la entrada aparece (SC-001: "sin intervención
   técnica").
3. `PATCH .../{id}` con `fields` distintos → `200`; verificar que una fila de
   `tenant.payment_methods` creada **antes** de este PATCH sigue con su `is_complete` sin cambios
   (Acceptance Scenario 2, edge case de `spec.md`).
4. `PATCH .../{id}` con `active=false` → `200`; verificar (US3 más abajo) que un tenant que lo
   tenía activo deja de verlo en `?available=true`, y que una venta histórica con ese método
   (`Payment`/`Sale` de fixture) no cambia (Acceptance Scenario 3 y 4).
5. Intentar los mismos endpoints sin `is_super_admin` (usuario `ADMIN` de tenant) → `403`.

```bash
python -m unittest app.characterization_tests.test_super_admin_payment_catalog -v
```

**Resultado esperado**: verde solo después de implementar el router de catálogo y el modelo
`PaymentMethodCatalog` (data-model.md).

## US2 — El Tenant Admin activa y configura métodos de pago

Fichero nuevo: `app/characterization_tests/test_sales_payment_methods_catalog.py`. Reemplaza los
casos de `test_sales_payment_methods.py` que asumían `name` libre (editado en el mismo commit,
citando esta spec).

1. Con el catálogo del paso US1 poblado (Efectivo/Nequi/Daviplata), `GET
   /sales/payment-methods/catalog` (tenant admin) → ve las tres, `already_activated=false`
   (Acceptance Scenario 1, [contracts/tenant-payment-methods.md](./contracts/tenant-payment-methods.md)).
2. `POST /sales/payment-methods` con `catalog_id` de Nequi y `payment_info={"celular":
   "3001234567"}` → `201`, `is_complete=true` (celular cumple `numeric(10)`) (Acceptance Scenario
   2).
3. `POST /sales/payment-methods` con `catalog_id` de Daviplata y **sin** `payment_info` → `201`,
   `is_complete=false`; `PATCH .../{id}` con `payment_info` incompleto/mal formado (ej. celular de
   3 dígitos) → sigue `is_complete=false`, `422` si el formato es inválido (Acceptance Scenario 3,
   FR-009).
4. `POST /sales/payment-methods` con `catalog_id` de Efectivo (sin `fields`) → `201`,
   `is_complete=true` de inmediato, sin pedir body adicional (Acceptance Scenario 4).
5. `PATCH .../{id}` con `active=false` sobre Nequi → `200`; verificar que sigue existiendo (no se
   borra) y que un `Payment` ya registrado con esa fila no cambia (Acceptance Scenario 5, Principio
   VII).
6. Repetir el paso 2 (activar Nequi otra vez) mientras la fila del paso 2 sigue `active=true` →
   `409` (FR-017, una sola configuración activa por `catalog_id`).
7. `POST /sales/payment-methods` con `catalog_id` de un método `active=false` en el catálogo →
   `409` (no se puede activar algo que el Super Admin desactivó).

```bash
python -m unittest app.characterization_tests.test_sales_payment_methods_catalog -v
python -m unittest app.characterization_tests.test_sales_payment_methods -v
```

**Resultado esperado**: `test_sales_payment_methods.py` verde con los casos actualizados a
`catalog_id` (ya no prueba `name`/`type` libres, que FR-007/FR-011 eliminan).

## US3 — El Cajero cobra usando solo métodos completamente disponibles

Fichero nuevo: `app/characterization_tests/test_sales_payment_methods_checkout.py`.

1. Tenant con "Efectivo" y "Nequi" (`active=true`, `is_complete=true`) y "Daviplata"
   (`active=true`, `is_complete=false`, del paso US2.3) → `GET
   /sales/payment-methods?available=true` → solo Efectivo y Nequi, **sin** `payment_info` en la
   respuesta (Acceptance Scenario 1, FR-012a, clarificación 2026-08-24 #1,
   [contracts/checkout-payment-methods.md](./contracts/checkout-payment-methods.md)).
2. Completar `payment_info` de Daviplata (formato válido) → `GET .../?available=true` inmediato →
   ahora incluye Daviplata (Acceptance Scenario 2, SC-002 "de inmediato").
3. Desactivar Daviplata en el catálogo (`PATCH /super-admin/payment-methods-catalog/{id}` con
   `active=false`) → `GET .../?available=true` → Daviplata ya no aparece, aunque
   `tenant.payment_methods.active` de esa fila siga en `true` (Acceptance Scenario 3, FR-013).
4. `GET /sales/payment-methods` (sin `available`) → sigue devolviendo las tres filas con
   `PaymentMethodResponse` completo (uso administrativo sin cambios) — confirma que el filtro nuevo
   no afecta la pantalla de configuración del tenant.

```bash
python -m unittest app.characterization_tests.test_sales_payment_methods_checkout -v
```

## Migración de datos existentes (FR-015/FR-015a)

No es un test de `characterization_tests` (es un script de un solo uso sobre datos reales) — se
valida por separado, contra una copia de la base de producción o un tenant de prueba con datos
representativos:

```bash
python -m app.scripts.migrate_payment_methods_catalog --report-only
# revisar el reporte; para cada nombre no reconocido, el Super Admin decide si lo agrega al
# catálogo vía POST /super-admin/payment-methods-catalog (US1)
python -m app.scripts.migrate_payment_methods_catalog
```

**Resultado esperado** (SC-004): cero filas `tenant.payment_methods` con `active=true` y
`catalog_id IS NULL` después de la segunda ejecución; cada tenant ve en `GET
/sales/payment-methods?available=true` exactamente los métodos que veía antes de la migración
(comparar contra un snapshot tomado antes del Paso 2 de data-model.md).

## Frontend (`pos-heladeria`)

```bash
cd ../pos-heladeria
npm test
```

Specs nuevos/editados relevantes: `payment-method-catalog.service.spec.ts` (nuevo, módulo
`super-admin`), `payment-method.service.spec.ts` (editado, módulo `sales`, ya existente). Validación
manual end-to-end (SC-001/SC-002, no automatizable sin navegador real):

1. Como Super Admin, crear "Daviplata" en `/super-admin/payment-methods-catalog` → confirmar que
   aparece en `/dashboard/ajustes/metodos-pago` de un tenant de prueba en menos de 5 minutos
   (SC-001).
2. Como Tenant Admin, activar Nequi y completar el celular → confirmar que aparece en el selector
   de método de pago del panel de cobro (`pos-checkout-panel.component.ts`) de inmediato, sin
   recargar sesión (SC-002).
