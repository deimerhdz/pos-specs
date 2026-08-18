# Quickstart: validar la corrección de zona horaria en el POS de staff (A-09)

Guía de ejecución para comprobar que la corrección cumple su contrato. No repite firmas ni tablas
ya detalladas en [data-model.md](./data-model.md) y
[contracts/promotions-list-endpoint.md](./contracts/promotions-list-endpoint.md) — solo enlaza a
ellas. Dos repos involucrados: `../pos-backend` (header nuevo) y `../pos-heladeria` (consumo del
header, corrección de los 4 puntos de invocación).

## Parte 1 — Backend (`../pos-backend`)

**Prerequisitos**: entorno virtual activado (Python 3.14).

```bash
cd ../pos-backend
source env/bin/activate
```

### Paso 1 — Confirmar que hoy no hay ninguna caracterización de `promotions/router.py`

```bash
ls app/characterization_tests/test_promotions_router.py 2>&1
# Resultado esperado antes del fix: "No such file or directory"
```

### Paso 2 — Escribir el test nuevo (RED, antes del fix)

Crear `app/characterization_tests/test_promotions_router.py`, invocando `list_promotions`
directamente como función Python (mismo patrón que `test_table_sessions_router.py` — `Depends(...)`
nunca se resuelve así):

```python
"""CONGELA comportamiento corregido: app/api/v1/promotions/router.py:37
(list_promotions) expone X-Server-Time — A-09 (registro-de-anomalias.md,
reapertura 2026-08-18).

Ejecutar solo este módulo:

    python -m unittest app.characterization_tests.test_promotions_router -v
"""
from datetime import datetime, timezone
import unittest
from types import SimpleNamespace
from unittest import mock

from fastapi import Response

from app.characterization_tests import cart_fixtures as fx
from app.api.v1.promotions import router as promotions_router


class TestListPromotionsA09(unittest.TestCase):
    def test_expone_x_server_time_en_utc(self):
        db = fx.new_session()
        user = SimpleNamespace(id="u1", tenant_id="t1")  # doble mínimo de get_current_user
        response = Response()

        instant = datetime(2026, 8, 18, 22, 30, 5, tzinfo=timezone.utc)
        with mock.patch("app.api.v1.promotions.router.datetime") as mocked:
            mocked.now.return_value = instant
            promotions_router.list_promotions(
                response=response, page=1, size=20, status_filter=None,
                search=None, db=db, _=user,
            )

        self.assertEqual(response.headers["X-Server-Time"], instant.isoformat())
```

```bash
python3 -m unittest app.characterization_tests.test_promotions_router -v
```

**Resultado esperado antes del fix**: falla — `list_promotions` no acepta hoy un parámetro
`response` ni escribe ningún header (`TypeError` o `KeyError` según el punto exacto del test).

### Paso 3 — Aplicar la corrección backend

Editar `app/api/v1/promotions/router.py` (research.md Decisión 1):

```python
# Imports nuevos (no existían en este fichero):
from datetime import datetime, timezone
from fastapi import Response  # se agrega Response al import ya existente de fastapi

@router.get("", response_model=Page[PromotionResponse], summary="Listar promociones")
def list_promotions(
    response: Response,
    page: int = Query(1, ge=1),
    size: int = Query(20, ge=1, le=100),
    status_filter: str | None = Query(None, alias="status"),
    search: str | None = Query(None, description="Búsqueda por nombre."),
    db: Session = Depends(get_db),
    _: User = Depends(get_current_user),
):
    response.headers["X-Server-Time"] = datetime.now(timezone.utc).isoformat()
    return paginate(db, service.list_query(search=search, status_filter=status_filter), page, size)
```

Editar `app/main.py:95` (research.md Decisión 1):

```python
expose_headers=["ETag", "Retry-After", "X-Server-Time"],
```

### Paso 4 — Confirmar la corrección backend

```bash
python3 -m unittest app.characterization_tests.test_promotions_router -v
```

**Resultado esperado tras el fix**: en verde (CA1 del lado servidor: el header viaja con la hora
UTC del servidor).

### Paso 5 — No regresión en el motor de promociones (A-07 protegida)

```bash
python3 -m unittest app.scripts.test_promotions_rules -v
```

**Resultado esperado**: sin cambios — este script (único en CI) ejercita
`active_discount_promotions`/`local_now`/`best_line_discount`, que esta spec no toca.

## Parte 2 — Frontend (`../pos-heladeria`)

**Prerequisitos**: `npm install` ya ejecutado.

```bash
cd ../pos-heladeria
```

### Paso 6 — Escribir el test de `PromotionService.now()`/`ready` (RED, antes del fix)

Crear (o ampliar) `src/app/modules/promotions/services/promotion.service.spec.ts`, mismo harness
que `product.service.spec.ts` (`provideHttpClientTesting`/`provideTanStackQuery`):

```typescript
it('antes del primer sync, ready() es false y now() no se usa (FR-004)', () => {
  expect(service.ready()).toBe(false);
});

it('tras el primer GET /promotions activo, now() usa el offset del servidor, no el reloj local', async () => {
  vi.setSystemTime(new Date('2026-08-18T22:30:00.000Z')); // reloj local del test
  service.loadActive();

  const req = http.expectOne((r) => r.url === PROMOTIONS && r.params.get('status') === 'active');
  req.flush(
    { items: [], total: 0, page: 1, size: 100, pages: 0 },
    { headers: { 'X-Server-Time': '2026-08-18T22:32:00.000Z' } }, // servidor 2 min adelantado
  );
  await tick();

  expect(service.ready()).toBe(true);
  expect(service.now().toISOString()).toBe('2026-08-18T22:32:00.000Z');
});
```

```bash
npm test -- promotion.service.spec.ts
```

**Resultado esperado antes del fix**: falla — `PromotionService` no tiene hoy `now()`/`ready`.

### Paso 7 — Aplicar la corrección en `PromotionService`

Editar `src/app/modules/promotions/services/promotion.service.ts` (research.md Decisiones 1/2/4):

```typescript
readonly serverTimeOffsetMs = signal<number | null>(null);
readonly ready = computed(() => this.serverTimeOffsetMs() !== null);

now(): Date {
  return new Date(Date.now() + (this.serverTimeOffsetMs() ?? 0));
}

private readonly activeQuery = injectPagedQuery<Promotion>({
  queryKey: () => ['promotions', 'active'],
  queryFn: () =>
    firstValueFrom(
      this.http.get<Page<Promotion>>(this.baseUrl, {
        params: { status: 'active', size: 100 },
        observe: 'response',
      }),
    ).then((res) => {
      const serverTime = res.headers.get('X-Server-Time');
      if (serverTime) this.serverTimeOffsetMs.set(new Date(serverTime).getTime() - Date.now());
      return res.body!;
    }),
  enabled: () => this.wantsActive(),
});
```

### Paso 8 — Escribir el test de los 4 puntos de invocación corregidos (RED, antes del fix)

Ampliar `src/app/modules/tables/services/pos-terminal.store.spec.ts` con un doble simple de
`PromotionService` (research.md Decisión 3 — sin `TestBed` completo del store):

```typescript
it('combos no evalúa vigencia con el reloj local si PromotionService aún no está ready (A-09)', () => {
  // doble mínimo: ready() = false, now() nunca debería llamarse
  // ... construir el `computed` de combos con el doble inyectado y verificar que devuelve []
});

it('combos usa promotionService.now(), no el reloj del sistema de pruebas (A-09)', () => {
  // reloj del sistema de pruebas fijado a un instante FUERA de la ventana de la promo;
  // promotionService.now() (doble) fijado a un instante DENTRO de la ventana;
  // se espera que el resultado sea "vigente" — solo posible si se usó el mock, no Date.now() real
});
```

```bash
npm test -- pos-terminal.store.spec.ts
```

**Resultado esperado antes del fix**: falla — los 4 sitios siguen usando `new Date()`.

### Paso 9 — Aplicar la corrección en `pos-terminal.store.ts`

Los 4 sitios (líneas 248, 262, 386, 1190), con la guarda de `ready()` (research.md Decisión 4):

```typescript
// Antes (A-09):
const now = new Date();
// Después:
if (!this.promotionService.ready()) return /* [] | new Map() | sin descuento, según el sitio */;
const now = this.promotionService.now();
```

### Paso 10 — Confirmar la corrección frontend

```bash
npm test -- promotion.service.spec.ts pos-terminal.store.spec.ts promotion-pricing.util.spec.ts
```

**Resultado esperado tras el fix**: los tres ficheros en verde — `promotion-pricing.util.spec.ts`
(sin cambios, funciones puras intactas) confirma que no hubo regresión en el cálculo en sí
(FR-005, A-10); los otros dos confirman la corrección (CA1/CA2/CA3/CA4).

## Verificación final — SC-001 a SC-006

```bash
# Backend
cd ../pos-backend && python3 -m unittest discover -s app/characterization_tests -p "test_*.py"

# Frontend
cd ../pos-heladeria && npm test
```

Ambas suites completas en verde, incluidos los ficheros nuevos/ampliados de esta spec y todos los
preexistentes sin cambios, es la señal de que la corrección está completa y no introdujo ninguna
regresión (Principio II) — en particular, que el motor de promociones del backend (A-07) y el
desempate del frontend (A-10) quedaron intactos.
