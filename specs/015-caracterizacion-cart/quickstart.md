# Quickstart: validar la red de characterization tests de `cart`

Guía para correr y verificar, de forma local, lo que esta spec entrega. No sustituye `tasks.md`
(fase de implementación) — asume que `test_cart_service.py`, `test_cart_router.py` y
`cart_fixtures.py` ya existen.

## Prerrequisitos

- Repositorio `pos-backend` (sibling de `pos-specs`) con su entorno Python activado
  (`../pos-backend/env/`, o `pip install -r requirements.txt` en un venv nuevo — sin dependencias
  adicionales a las ya congeladas en el repo, Principio IV).
- No hace falta Postgres real ni Redis real corriendo: toda la suite corre contra SQLite en memoria y
  dobles de Redis (ver `research.md` §1-2).

```bash
cd ../pos-backend
python -m venv env  # si no existe ya
source env/bin/activate
pip install -r requirements.txt
```

## Escenario 1 — Historia 1 en aislamiento (las 11 funciones de `service.py`)

```bash
python -m unittest app.characterization_tests.test_cart_service -v
```

**Resultado esperado**: todos los tests en verde, sin que existan aún `test_cart_router.py` ni el
wiring de CI — confirma el "Independent Test" de la Historia 1 en `spec.md`. Debe incluir al menos un
test cuyo nombre/docstring cite A-08 y otro que cite A-17 (R16) (SC-002).

## Escenario 2 — Historia 2 en aislamiento (los 9 endpoints de `router.py`)

```bash
python -m unittest app.characterization_tests.test_cart_router -v
```

**Resultado esperado**: todos los tests en verde, reutilizando los fixtures de la Historia 1
(`cart_fixtures.py`) para construir el `SessionContext` que cada endpoint necesita. Debe incluir el
caso de `POST /cart/leave` sin token (204, nunca 401) y el caso de `POST /cart/items` sobre el límite
de tasa (429 + `Retry-After`).

## Escenario 3 — determinismo (SC-003)

```bash
for i in 1 2 3; do
  python -m unittest discover -s app/characterization_tests -p 'test_*.py' > /tmp/run_$i.log 2>&1
done
diff /tmp/run_1.log /tmp/run_2.log && diff /tmp/run_2.log /tmp/run_3.log && echo "IDÉNTICOS"
```

**Resultado esperado**: `IDÉNTICOS` — mismos tests en verde en las tres corridas, sin flakiness
(ningún test depende del reloj real, de Redis, de Postgres real, ni de orden de ejecución).

## Escenario 4 — la red detecta un cambio real (Historia 3, acceptance scenario 3)

Prueba deliberada de que la suite no es "un test que siempre pasa": modificar temporalmente una línea
de `cart/service.py` (por ejemplo, invertir una condición en `cancel_my_order`), correr la suite, y
confirmar que al menos un test falla en rojo. Revertir el cambio inmediatamente después — esta spec no
autoriza ningún cambio real en `cart/service.py` ni `cart/router.py` (FR-012).

```bash
git diff app/api/v1/cart/service.py  # confirmar que el cambio de prueba es visible
python -m unittest app.characterization_tests.test_cart_service -v  # al menos 1 rojo
git checkout -- app/api/v1/cart/service.py  # revertir antes de continuar
```

## Escenario 5 — verificar el wiring de CI (SC-004)

No requiere abrir un PR real para verificar localmente el mismo comando que CI ejecuta:

```bash
python -m app.scripts.test_promotions_rules   # paso existente, sin reemplazar (FR-009)
python -m unittest discover -s app/characterization_tests -p 'test_*.py' -v  # paso nuevo
```

Para la verificación real en CI: abrir un PR trivial contra la rama de esta spec y confirmar en la
ejecución de GitHub Actions que ambos pasos del job `test` corren y pasan
(`.github/workflows/deploy.yml`).

## Qué NO valida esta guía

- No valida reglas propias de promociones ni de checkout (fuera de alcance, ver Assumptions de
  `spec.md`) — solo que `cart` las consume tal como hoy las consume.
- No es una guía de extracción: esta spec es characterization pura (Principio III). Cuando exista una
  spec futura de extracción de `cart`, esa suite es la que sirve de línea base — no se regenera aquí.
