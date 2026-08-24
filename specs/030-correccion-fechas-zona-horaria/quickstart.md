# Quickstart: validar la corrección global de fechas, horas y zonas horarias

Guía de ejecución para comprobar que la corrección cumple su contrato. No repite firmas ni tablas ya
detalladas en [data-model.md](./data-model.md) y `contracts/` — solo enlaza a ellas. Dos repos
involucrados: `../pos-backend` (mecanismo central, migración, filtros) y `../pos-heladeria`
(`TenantDatePipe`, consumo del campo `timezone`). Se muestra el patrón aplicado a Ventas (defecto
reportado, P1); el mismo patrón, con los ficheros listados en `plan.md` → Project Structure, aplica al
resto de las once entidades.

## Parte 1 — Backend (`../pos-backend`)

**Prerequisitos**: entorno virtual activado (Python 3.14).

```bash
cd ../pos-backend
source env/bin/activate
```

### Paso 1 — Mecanismo central: `app/core/timezone.py` (research.md Decisiones 1-2)

Crear el módulo con `resolve_timezone`, `UtcDatetime`, `local_day_bounds_utc`, `utc_now` (ver
`data-model.md` para las firmas exactas). Sin este módulo nada más de esta spec puede escribirse —
es la base de la que dependen los pasos siguientes.

### Paso 2 — Migración: `Tenant.timezone`

```bash
alembic revision --autogenerate -m "add_tenant_timezone"
# revisar el archivo generado: server_default='America/Bogota', nullable=False
alembic upgrade head
```

**Verificación**:

```bash
python3 -c "
from app.core.db import with_db
from app.core.models import Tenant
with with_db(None) as db:
    t = db.query(Tenant).first()
    print(t.timezone)  # esperado: America/Bogota, para cualquier tenant existente
"
```

### Paso 3 — Escribir el test de Ventas (RED, antes del fix)

`app/characterization_tests/test_sales_timezone.py` (nuevo — cumple `FR-010` para la entidad Ventas):

```python
"""CONGELA comportamiento corregido: sales/schemas.py:SaleResponse.sold_at
viaja con offset UTC explícito y sales/service.py filtra por medianoche de
Bogotá, no UTC — A-50 (registro-de-anomalias.md).

    python -m unittest app.characterization_tests.test_sales_timezone -v
"""
import unittest
from datetime import date, datetime, timezone

from app.characterization_tests import cart_fixtures as fx
from app.api.v1.sales import schemas as sales_schemas
from app.api.v1.sales import service as sales_service


class TestSaleSoldAtUtcSerialization(unittest.TestCase):
    def test_sold_at_lleva_offset_utc_explicito(self):
        # instante naive tal como lo devuelve SQLAlchemy hoy (ya es UTC)
        naive = datetime(2026, 8, 24, 12, 53, 7)
        resp = sales_schemas.SaleResponse.model_construct(sold_at=naive, ...)
        self.assertTrue(resp.model_dump(mode="json")["sold_at"].endswith("+00:00"))


class TestSalesFiltroMedianocheBogota(unittest.TestCase):
    def test_venta_23_59_bogota_incluida_en_filtro_del_dia(self):
        # 23:59 hora de Bogota del 24/08 = 04:59 UTC del 25/08
        sold_at_utc = datetime(2026, 8, 25, 4, 59, 0)
        db = fx.new_session()
        # ... crear Sale con sold_at=sold_at_utc, tenant con timezone='America/Bogota'
        stmt = sales_service.build_sales_query(
            tenant=fx.tenant_bogota(), date_from=date(2026, 8, 24), date_to=date(2026, 8, 24),
        )
        # ... assert que la venta aparece en el resultado
```

```bash
python3 -m unittest app.characterization_tests.test_sales_timezone -v
```

**Resultado esperado antes del fix**: falla — `sold_at` no lleva offset y `build_sales_query` no
acepta `tenant`.

### Paso 4 — Aplicar la corrección en Ventas

- `sales/schemas.py:134`: `sold_at: datetime` → `sold_at: UtcDatetime` (import desde
  `app.core.timezone`).
- `sales/schemas.py:94-100` (`PaymentResponse`): agregar `paid_at: UtcDatetime` (research.md
  Decisión 9).
- `sales/service.py:190-216` (`build_sales_query`): agregar parámetro `tenant: Tenant`, reemplazar la
  comparación cruda por `local_day_bounds_utc` (ver `contracts/date-range-filters.md`).
- `sales/router.py`: pasar `tenant` (ya resuelto por `Depends(get_tenant)`) a `build_sales_query`.

### Paso 5 — Confirmar la corrección backend

```bash
python3 -m unittest app.characterization_tests.test_sales_timezone -v
```

**Resultado esperado tras el fix**: en verde — CA1/CA2 de Historia 1, CA1/CA2/CA3 de Historia 3 para
Ventas.

### Paso 6 — Test de valor histórico sin cambios (`FR-010`, Historia 6)

```python
"""CONGELA comportamiento actual: el valor almacenado de una venta ya
existente no cambia por esta corrección — solo su serialización/filtrado.
"""
def test_valor_almacenado_no_cambia(self):
    db = fx.new_session()
    sale = fx.crear_venta(sold_at=datetime(2026, 8, 20, 15, 0, 0))
    antes = sale.sold_at
    # ... ejecutar cualquier ruta de lectura/serialización introducida por esta spec
    db.refresh(sale)
    self.assertEqual(sale.sold_at, antes)  # el valor crudo en DB es idéntico
```

### Paso 7 — Repetir el patrón para el resto de entidades

Mismo patrón (schema → `UtcDatetime`, test de serialización, test de medianoche cuando aplique
filtro) para Órdenes, Pagos, Caja, Inventario — mínimo un test por entidad (`FR-010`). Ver
`plan.md` → Project Structure para el fichero exacto de cada una; `data-model.md` para la tabla
completa de las once entidades.

### Paso 8 — Zona horaria por tenant (Historia 4)

```bash
python3 -m app.scripts.set_tenant_timezone --host otro-tenant.skeilopos.com --timezone America/Mexico_City
```

**Verificación**: un valor inválido se rechaza antes de persistirse:

```bash
python3 -m app.scripts.set_tenant_timezone --host otro-tenant.skeilopos.com --timezone No/Existe
# Resultado esperado: error explícito (ZoneInfoNotFoundError capturado), sin escritura en DB
```

### Paso 9 — No regresión en el motor de promociones (A-07 protegida)

```bash
python3 -m unittest app.scripts.test_promotions_rules -v
```

**Resultado esperado**: sin cambios — el cambio aditivo de `_tz()` (research.md Decisión 10) no
altera ningún resultado para el tenant único que existe hoy.

### Paso 10 — Registro de anomalías (`FR-011`)

Agregar entrada `A-50` (siguiente disponible tras `A-49`) y actualizar `A-46` en
`../pos-specs/specs/000-reconocimiento/registro-de-anomalias.md`, siguiendo el mismo formato de las
entradas existentes (`### A-NN — <título>`, `**Descripción**`, `- **CÓDIGO**:`, `**Clasificación**:`,
`**Depende de esto**:`, `**Tratamiento acordado**:`).

## Parte 2 — Frontend (`../pos-heladeria`)

**Prerequisitos**: `npm install` ya ejecutado.

```bash
cd ../pos-heladeria
```

### Paso 11 — `TenantInfo.timezone` (research.md, contrato `tenant-info-endpoint.md`)

Editar `tenant-info.service.ts:8-16`: agregar `timezone: string` a la interfaz `TenantInfo`. Sin
cambios de wiring — `load()` ya trae el campo nuevo dentro de la respuesta de `GET /tenant`.

### Paso 12 — Escribir el test de `TenantDatePipe` (RED, antes del fix)

`src/app/shared/pipes/tenant-date.pipe.spec.ts` (nuevo):

```typescript
it('formatea en la zona horaria del tenant, no en la del navegador (A-50)', () => {
  tenantInfo.info.set({ ...tenantInfoFixture, timezone: 'America/Bogota' });
  // 2026-08-24T12:53:07Z = 2026-08-24 07:53 hora de Bogotá
  const resultado = pipe.transform('2026-08-24T12:53:07.000Z', 'dd/MM/yyyy HH:mm');
  expect(resultado).toBe('24/08/2026 07:53');
});

it('usa America/Bogota como respaldo si el tenant aún no cargó', () => {
  tenantInfo.info.set(null);
  const resultado = pipe.transform('2026-08-24T12:53:07.000Z', 'HH:mm');
  expect(resultado).toBe('07:53');
});
```

```bash
npm test -- tenant-date.pipe.spec.ts
```

**Resultado esperado antes del fix**: falla — `TenantDatePipe` no existe.

### Paso 13 — Crear `TenantDatePipe` y `date-format.util.ts`

Ver `research.md` Decisiones 6-7 para la implementación exacta (`DatePipe` nativo + tercer argumento
de zona horaria; `businessToday()` con `Intl.DateTimeFormat`).

```bash
npm test -- tenant-date.pipe.spec.ts
```

**Resultado esperado tras el fix**: en verde.

### Paso 14 — Aplicar el pipe en Ventas (defecto reportado)

Editar `sales-page.component.ts:108,155`:

```html
<!-- Antes -->
{{ s.sold_at | date: 'dd/MM/yyyy HH:mm' }}
<!-- Después -->
{{ s.sold_at | tenantDate: 'dd/MM/yyyy HH:mm' }}
```

Verificación manual (opcional, navegador con reloj/zona distinta a Bogotá): crear una venta a las
07:53 hora de Bogotá, confirmar que el listado y el modal de recibo muestran `07:53`, no la hora UTC
cruda ni la hora local del navegador.

### Paso 15 — Aplicar el mismo pipe al resto de sitios `| date` y al formateador de `cash-session.store.ts`

5 sitios `| date` (`admin-dashboard.component.ts:123`, `inventory-page.component.ts:212,297`,
`order-detail.component.ts:64`, `orders-page.component.ts:92`) → `| tenantDate`. 4 métodos de
`cash-session.store.ts` (`fmtTime`/`fmtDate`, líneas 598-625) → inyectar `TenantDatePipe` y llamar
`.transform(iso, formato)` en vez de `new Date(iso).toLocaleString(...)`.

```bash
npm test -- cash-session.store.spec.ts
```

### Paso 16 — `reports.service.ts::getDateRange` (Historia 3, research.md Decisión 7)

```bash
npm test -- reports.service.spec.ts
```

**Resultado esperado antes del fix**: si existe un test que fija "hoy" contra `new Date()` del
navegador sin zona, revisar que no quede en rojo por asumir un huso horario de ejecución — ajustarlo
para inyectar `TenantInfoService` con `timezone: 'America/Bogota'` fijo, y comparar contra
`businessToday('America/Bogota')`, no contra el reloj del entorno de CI (que puede no estar en
UTC-5).

### Paso 17 — Confirmar todo el frontend

```bash
npm test -- tenant-date.pipe.spec.ts cash-session.store.spec.ts reports.service.spec.ts sales-page.component.spec.ts
```

## Verificación final — SC-001 a SC-006

```bash
# Backend
cd ../pos-backend && python3 -m unittest discover -s app/characterization_tests -p "test_*.py"

# Frontend
cd ../pos-heladeria && npm test
```

Ambas suites completas en verde, incluidos los ficheros nuevos/ampliados de esta spec y todos los
preexistentes sin cambios, es la señal de que la corrección está completa:

- **SC-001/SC-002**: los tests de serialización por entidad (Paso 7) demuestran offset UTC explícito
  y el mismo mecanismo (`UtcDatetime`/`TenantDatePipe`) en las once entidades.
- **SC-003**: los tests de medianoche (Paso 3/6) para Ventas y Reportes.
- **SC-004**: el test de valor histórico sin cambios (Paso 6).
- **SC-005**: el Paso 8 (dos tenants, dos zonas horarias, sin afectarse entre sí).
- **SC-006**: revisión manual — ningún `+ timedelta(hours=...)`/`- timedelta(hours=...)` ni ajuste
  manual de horas queda en el código tocado por esta spec (búsqueda `grep -rn "timedelta(hours"` en
  los ficheros listados en `plan.md` → Project Structure, comparado contra el estado previo al fix).
- El motor de promociones (A-07, Paso 9) y el desempate del frontend (A-10, sin cambios) quedan
  intactos.
