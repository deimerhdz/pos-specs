# Quickstart: validar la red de characterization tests de `table_sessions`

Guía para correr y verificar, de forma local, lo que esta spec entrega. No sustituye `tasks.md`
(fase de implementación) — asume que `test_table_sessions_split_blindaje.py`,
`test_table_sessions_service.py`, `test_table_sessions_router.py` y
`table_sessions_fixtures.py` ya existen.

## Prerrequisitos

- Repositorio `pos-backend` (sibling de `pos-specs`) con su entorno Python activado
  (`../pos-backend/env/`, o `pip install -r requirements.txt` en un venv nuevo — sin dependencias
  adicionales a las ya congeladas en el repo, Principio IV).
- No hace falta Postgres real corriendo: toda la suite corre contra SQLite en memoria. No hace
  falta Redis real tampoco: `bill_changed` se intercepta explícitamente (FR-010a) y los otros tres
  eventos de `close_session` son fail-open sin Redis disponible (research.md §3) — si tu entorno
  local sí tiene un Redis real levantado en `REDIS_URL`, esos tres se publicarán de verdad, lo cual
  no afecta ningún resultado de test.

```bash
cd ../pos-backend
python -m venv env  # si no existe ya
source env/bin/activate
pip install -r requirements.txt
```

## Escenario 1 — Historia 1 en aislamiento (A-15 [PROTEGIDA], split blindaje)

```bash
python -m unittest app.characterization_tests.test_table_sessions_split_blindaje -v
```

**Resultado esperado**: todos los tests en verde, sin que existan aún los otros dos ficheros de
esta spec ni ningún cambio de CI — confirma el "Independent Test" de la Historia 1 en `spec.md`.
Debe incluir al menos 4 casos citando A-15 explícitamente en el nombre/docstring, uno por cada
hueco de seguridad cerrado (comensal repetido, importes de raíz en modo split, bloque sin
comensal sin nombre, cobertura exacta comensal-consumo ↔ comensal-split) — es, de las cinco
anomalías de esta spec, la que más casos concentra (SC-002).

## Escenario 2 — Historia 2 en aislamiento (las 9 funciones de `service.py`)

```bash
python -m unittest app.characterization_tests.test_table_sessions_service -v
```

**Resultado esperado**: todos los tests en verde. Debe incluir al menos un test citando cada una
de A-01, A-17 (R12), A-29 y A-38 (RN-MESA-13 y RN-MESA-24) en su nombre/docstring (SC-002), y
cubrir las 9 funciones públicas (`get_session`, `has_billable_orders`, `try_release_if_empty`,
`list_sessions`, `compute_bill`, `close_session`, `add_participant`, `remove_participant`,
`set_assignments` — SC-001).

## Escenario 3 — Historia 3 en aislamiento (los 7 endpoints de `router.py`)

```bash
python -m unittest app.characterization_tests.test_table_sessions_router -v
```

**Resultado esperado**: todos los tests en verde, reutilizando `table_sessions_fixtures.py` para
construir los dobles de `Tenant`/`User` que cada endpoint necesita. Cubre los 7 endpoints
(`GET /table-sessions`, `GET /table-sessions/{id}`, `POST .../participants`,
`DELETE .../participants/{id}`, `PUT .../assignments`, `GET .../bill`, `POST .../close` — SC-001).

## Escenario 4 — determinismo (SC-003)

```bash
for i in 1 2 3; do
  python -m unittest discover -s app/characterization_tests -p 'test_*.py' > /tmp/run_$i.log 2>&1
done
diff /tmp/run_1.log /tmp/run_2.log && diff /tmp/run_2.log /tmp/run_3.log && echo "IDÉNTICOS"
```

**Resultado esperado**: `IDÉNTICOS` — mismos tests en verde en las tres corridas, incluyendo los
de `cart` y el resto de `app/characterization_tests/` ya existentes, sin flakiness (ningún test
depende del reloj real, de Redis, de Postgres real, ni de orden de ejecución).

## Escenario 5 — la red detecta un cambio real (Historia 3, acceptance scenario 3)

Prueba deliberada de que la suite no es "un test que siempre pasa": modificar temporalmente una
línea de `table_sessions/service.py` (por ejemplo, invertir la condición de `_assert_closable` o
comentar el chequeo de comensales repetidos en `_close_split`), correr la suite, y confirmar que
al menos un test falla en rojo. Revertir el cambio inmediatamente después — esta spec no autoriza
ningún cambio real en `table_sessions/service.py` ni `router.py` (FR-014).

```bash
git diff app/api/v1/table_sessions/service.py  # confirmar que el cambio de prueba es visible
python -m unittest app.characterization_tests.test_table_sessions_split_blindaje -v  # al menos 1 rojo
git checkout -- app/api/v1/table_sessions/service.py  # revertir antes de continuar
```

## Escenario 6 — verificar que CI ya cubre los ficheros nuevos, sin tocar el workflow (SC-004)

A diferencia de `specs/015-caracterizacion-cart/`, esta spec **no modifica**
`.github/workflows/deploy.yml`: el paso que ya instala `requirements.txt` completo y descubre
`test_*.py` bajo `app/characterization_tests/` quedó instalado por esa spec anterior. Verificación
local del mismo comando que CI ejecuta:

```bash
python -m app.scripts.test_promotions_rules   # paso existente, sin tocar
python -m unittest discover -s app/characterization_tests -p 'test_*.py' -v  # ya incluye los 3 nuevos
```

Para la verificación real en CI: abrir un PR trivial contra la rama de esta spec y confirmar en la
ejecución de GitHub Actions que el job `test` recoge y pasa los tres ficheros nuevos sin que
`.github/workflows/deploy.yml` tenga ningún diff.

## Qué NO valida esta guía

- No valida reglas propias de `orders.checkout`, `promotions` ni `sales.builder`/`invoices` (fuera
  de alcance, ver Assumptions de `spec.md`) — solo que `table_sessions` las consume tal como hoy
  las consume.
- No es una guía de extracción: esta spec es characterization pura (Principio III). Cuando exista
  una spec futura de extracción de `table_sessions`, esta suite es la que sirve de línea base — no
  se regenera aquí.
