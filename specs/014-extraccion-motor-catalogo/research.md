# Phase 0 Research: Extracción del motor de catálogo

Esta spec no tiene incógnitas de "qué tecnología usar" en el sentido habitual — el código, el
lenguaje, el framework de test y las siete rutas consumidoras ya existen y están enumerados en la
propia spec. Las tres preguntas de la sesión de Clarifications (2026-08-17) ya quedaron resueltas
dentro de `spec.md`. Lo que queda para esta fase es investigar y decidir el **cómo** de la
extracción a nivel de código: estructura interna del paquete, mecánica de las fachadas, y la
herramienta concreta para el generador determinista de la Historia 2. Cada decisión se verificó
leyendo el código real de `pos-backend` (no se asume nada no confirmado en el repositorio).

## Decisión 1 — Ruta y nombre del paquete de destino

**Decision**: `app/catalog_engine/`, al mismo nivel que `app/models/` y `app/core/`.

**Rationale**: es la ruta que la propia spec anticipa en Assumptions ("`app/catalog_engine/` es
la ruta de destino salvo que la convención real del proyecto indique otra"). Se verificó
(`ls app/` en `pos-backend`) que no existe ya ningún paquete `catalog_engine` ni ningún otro
paquete con sufijo `_engine` — no hay colisión de nombre ni precedente que sugiera una convención
distinta. `app/` ya contiene paquetes de dominio a ese mismo nivel (`api/`, `core/`, `models/`),
así que un paquete nuevo ahí es consistente con la organización existente, no una excepción.

**Alternatives considered**:
- Anidarlo bajo `app/api/v1/catalog/engine/`: rechazado — mezclaría el paquete "puro,
  reutilizable en teoría por cualquier capa" con `app/api/v1/`, que es específicamente la capa de
  endpoints HTTP (`router.py`, `schemas.py` viven ahí). El motor no es un endpoint ni depende de
  FastAPI más allá de `HTTPException`/`status`, que ya usa hoy como mecanismo de error de
  dominio — no hay razón para acoplarlo estructuralmente a `api/v1`.
- Nombrarlo `app/catalog/` a secas: rechazado — colisionaría conceptualmente con
  `app/api/v1/catalog/` (incluye `router.py`, `schemas.py`, `service.py` de SKU, que no se tocan
  en esta spec) y generaría ambigüedad en imports (`app.catalog` vs `app.api.v1.catalog`).
  `catalog_engine` es inequívoco y coincide con el nombre que la spec ya usa en su título.

## Decisión 2 — División interna del paquete (núcleo vs. adaptadores)

**Decision**: cuatro ficheros — `__init__.py` (API pública), `core.py` (núcleo puro),
`pricing.py` (los cuatro adaptadores ex-`line_pricing.py`), `consumption.py` (los siete
adaptadores ex-`consumption_plan.py`).

**Rationale**: FR-003 exige que el núcleo puro "no importe `sqlalchemy` ni ningún módulo que a su
vez dependa de él", y SC-006 exige que esto sea "verificable por inspección estática". Aislar
`compute_line_price`, `_exige_maximo` y el dataclass `ConsumptionLine` en un fichero propio
(`core.py`) hace esa verificación trivial: `grep -n sqlalchemy app/catalog_engine/core.py` debe
devolver vacío. Mezclarlos en un único fichero junto a las once funciones que sí usan `db:
Session` obligaría a inspeccionar función por función en vez de fichero por fichero, que es
justo el tipo de ambigüedad que SC-006 quiere evitar. Mantener `pricing.py` y `consumption.py`
como dos ficheros (no fusionarlos en uno) conserva el agrupamiento original de los dos ficheros
legado, lo que mantiene el diff conceptualmente legible (cada función adaptadora migra al fichero
"equivalente", no a uno nuevo sin relación con su origen) sin que ninguna historia de usuario lo
exija de otro modo.

**Alternatives considered**:
- Un solo fichero `adapters.py` para las once funciones: rechazado por la razón de legibilidad de
  arriba — no aporta nada que la spec pida, y diluye la trazabilidad "esta función venía de
  `line_pricing.py`" vs. "venía de `consumption_plan.py`".
- Separar cada una de las 13 funciones en su propio fichero: rechazado — sobre-fragmentación no
  solicitada por ningún requisito; el edge case de la spec ("¿alguna función adaptadora tiene
  lógica pura mezclada que valdría separar más?") está explícitamente fuera de alcance ("no
  autoriza descomponer ninguna función existente en piezas más pequeñas").

## Decisión 3 — Ubicación del dataclass `ConsumptionLine`

**Decision**: vive en `core.py`, junto a las dos funciones puras.

**Rationale**: es un `@dataclass(frozen=True)` sin ningún import de `sqlalchemy` ni comportamiento
de I/O — es, por definición, parte del "núcleo puro" en el mismo sentido que FR-003 usa el
término, aunque FR-003 solo menciona explícitamente `compute_line_price` y `_exige_maximo`.
Colocarlo en `core.py` evita un import circular entre `pricing.py`/`consumption.py` (que no se
necesitan importar mutuamente hoy) y mantiene una única fuente de verdad para todo lo que no
depende de `db: Session`.

**Alternatives considered**:
- Un fichero `types.py` dedicado solo al dataclass: rechazado — es un único tipo de 3 campos: no
  justifica un fichero propio (regla de "no diseñar para lo hipotético").
- Dejarlo en `consumption.py` (junto a las funciones que lo producen, como en el código legado):
  rechazado — rompería la garantía de "cero imports de sqlalchemy" del fichero que efectivamente
  usan los otros dos ficheros del núcleo si en el futuro `core.py` necesitara importarlo (hoy no
  lo necesita, pero mantenerlo en `core.py` deja la frontera núcleo/adaptador basada en
  contenido real —qué importa cada fichero—, no en "de qué fichero legado vino".

## Decisión 4 — Mecánica de la fachada (Historia 3)

**Decision**: `line_pricing.py` y `consumption_plan.py` quedan reducidos a imports explícitos
desde `app.catalog_engine` con reexport nombrado (`from app.catalog_engine import X as X`, o
equivalente con `__all__`), preservando exactamente el reexport actual de
`line_pricing.py:31-36` (`ConsumptionLine`, `load_variant_groups`, `plan_line_consumption`,
`required_consumption`) del que depende `cart/service.py:31-36`.

**Rationale**: FR-008 exige que las fachadas "no contengan lógica propia de cálculo, validación o
consulta" — un `import ... as ...` por símbolo es la forma más directa de cumplir eso sin
ambigüedad. FR-009 exige explícitamente que el reexport de las líneas 31-36 siga funcionando
igual; replicar la misma mecánica (import con alias, no una función wrapper) es lo que garantiza
que `cart/service.py` no note ninguna diferencia — porque a nivel de bytecode/runtime un símbolo
reexportado así es indistinguible de estar definido ahí, que es literalmente lo que FR-010/SC-004
piden verificar (diff vacío en los siete consumidores).

**Alternatives considered**:
- Fachada por `from app.catalog_engine import *`: rechazado — importar con `*` no permite
  controlar explícitamente qué símbolos se reexportan según linters estáticos (requiere
  `__all__` para no disparar warnings de "unused import" en cada símbolo, y sigue sin dar el
  control fino que FR-001/FR-002 piden — nombre exacto, mismo símbolo).
- Funciones wrapper que llaman a las del paquete nuevo (`def compute_line_price(...): return
  catalog_engine.compute_line_price(...)`): rechazado — FR-008 dice explícitamente "sin contener
  lógica propia"; un wrapper con firma repetida es justo el tipo de lógica duplicada (y
  divergente si alguien edita solo un lado) que una fachada de reexport puro evita.

## Decisión 5 — Herramienta para el generador determinista de la Historia 2

**Decision**: `random.Random(seed)` de la biblioteca estándar de Python.

**Rationale**: la Constitución (Principio IV) dice explícitamente "cuando exista una solución
equivalente en la biblioteca estándar [...], esta se prefiere sobre una dependencia externa". El
requisito de la Historia 2 (FR-013, y la clarificación de sesión 2026-08-17) es concreto y
acotado: generar entre 100 y 200 combinaciones deterministas de variantes/grupos/recetas/niveles
de stock a partir del fixture ya sembrado por `app/characterization_tests/fixtures.py`, con
reproducibilidad byte a byte entre corridas de la misma semilla. `random.Random(seed).choice(...)`
/ `.sample(...)` sobre las listas de entidades ya creadas por el fixture cubre esto sin ningún
paquete adicional — no hace falta un motor de testing basado en propiedades (`hypothesis`) porque
el espacio de combinaciones no es infinito ni requiere shrinking: es una enumeración acotada sobre
datos ya conocidos de antemano.

**Alternatives considered**:
- `hypothesis` (property-based testing): rechazado — resolvería un problema más general (shrinking
  automático de casos que fallan, generación de datos arbitrarios) que esta spec no tiene: los
  "datos" ya existen en el fixture reutilizado, y lo único que hace falta es una combinatoria
  determinista sobre ellos. Añadirlo exigiría justificación y aprobación explícita
  (Constitución, Principio IV) para un problema que la biblioteca estándar ya resuelve.
- `random` del módulo global (sin instanciar `Random(seed)`) con `random.seed(seed)`: rechazado
  — muta estado global de proceso, lo que puede interferir con cualquier otro test que use
  `random` en la misma sesión de `unittest` (los characterization tests corren en el mismo
  proceso). Una instancia `Random(seed)` propia es autocontenida y no tiene ese riesgo de fuga
  entre tests.

## Decisión 6 — Convención de ejecución de tests

**Decision**: la Historia 2 se ejecuta con `python3 -m unittest`, igual que el resto del paquete
`app/characterization_tests/` (no se introduce `pytest`).

**Rationale**: se verificó (`find . -iname pytest.ini`, `cat pyproject.toml`) que `pos-backend` no
tiene configuración de `pytest` en ningún sitio, y que todos los characterization tests existentes
usan `unittest.TestCase` con `python3 -m unittest` (documentado explícitamente en
`golden_master/README.md`: "`python3 -m unittest app.characterization_tests.test_golden_master...
-v`"). Introducir `pytest` solo para el nuevo test rompería esa uniformidad sin ningún beneficio
que la spec pida, y contaría como dependencia nueva a justificar bajo el Principio IV sin
necesidad real.

**Alternatives considered**: `pytest` — rechazado por la razón de arriba; no hay ningún requisito
funcional que `unittest` no pueda cumplir (parametrización de 100-200 casos generados es
perfectamente expresable como un bucle `for caso in casos: with self.subTest(caso=caso): ...`
dentro de un único `test_...` de `unittest`).

## Resumen — incógnitas resueltas

No queda ningún `NEEDS CLARIFICATION` pendiente en el Technical Context de `plan.md`: lenguaje,
dependencias, storage, testing, plataforma, tipo de proyecto, y alcance ya estaban determinados
por el código existente y las tres respuestas de la sesión de Clarifications de `spec.md`. Esta
fase solo añadió las seis decisiones de diseño de arriba, ninguna de las cuales requiere
investigación externa adicional.
