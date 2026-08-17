# Phase 0 Research: Extracción del motor de stock de inventario

Esta spec no tiene incógnitas de "qué tecnología usar" — el código, el lenguaje, el framework de
test y los cuatro ficheros consumidores ya existen y están enumerados en la propia spec. La única
pregunta de la sesión de Clarifications (2026-08-17, dimensionamiento de la batería de la Historia
2) ya quedó resuelta dentro de `spec.md`. Lo que queda para esta fase es investigar y decidir el
**cómo** de la extracción a nivel de código, y resolver explícitamente FR-009 (golden master vs.
revisión manual), que `spec.md` deja como decisión de planificación. Cada decisión se verificó
leyendo el código real de `pos-backend` (no se asume nada no confirmado en el repositorio).

## Decisión 1 — Ruta y nombre del paquete de destino

**Decision**: `app/inventory_engine/`, al mismo nivel que `app/models/`, `app/core/` y
`app/catalog_engine/`.

**Rationale**: es la ruta que la propia spec anticipa en Assumptions ("`app/inventory_engine/` es
la ruta de destino, siguiendo la misma convención de nombrado que `app/catalog_engine/`"). Se
verificó (`find app -iname "*inventory_engine*"` en `pos-backend`) que no existe colisión de
nombre. La spec 014 ya sentó el precedente de paquetes `*_engine` a ese nivel — seguir la misma
convención evita introducir una segunda forma de nombrar módulos extraídos en el mismo repositorio.

**Alternatives considered**:
- Anidarlo bajo `app/api/v1/inventory/engine/`: rechazado por la misma razón que la spec 014
  rechazó el equivalente para catálogo — mezclaría el paquete con `app/api/v1/`, la capa
  específica de endpoints HTTP (`router.py`, `schemas.py`). El motor no depende de FastAPI en
  absoluto (ni siquiera de `HTTPException` directamente — `InsufficientStockError` ya la
  encapsula en `app.core.exceptions`), así que hay aún menos razón para acoplarlo a `api/v1`.
- Nombrarlo `app/inventory/` a secas: rechazado — colisionaría conceptualmente con
  `app/api/v1/inventory/` (incluye `router.py`, `schemas.py`, `service.py`, que no se tocan salvo
  el import mínimo de FR-008) y generaría ambigüedad en imports.

## Decisión 2 — División interna del paquete (sin frontera núcleo/adaptador)

**Decision**: dos ficheros — `__init__.py` (API pública) y `stock.py` (las tres funciones, sin
dividir).

**Rationale**: a diferencia de `catalog_engine/` (spec 014), donde FR-003 exigía aislar dos
funciones puras sin `sqlalchemy` en un `core.py` propio verificable por `grep`, esta spec no tiene
ningún requisito equivalente. Se verificó leyendo `app/api/v1/inventory/stock.py` completo: las
tres funciones (`lock_items`, `record_movement`, `apply_adjustment`) reciben `db: Session` y
ejecutan `.with_for_update()` — ninguna es pura. FR-002 exige preservar ese patrón "exactamente en
los mismos puntos [...] sin introducir una abstracción nueva sobre él", lo que apunta en la
dirección contraria a fragmentar el fichero: menos ficheros, menos superficie donde el orden de
bloqueo se pueda alterar por accidente al mover código. Mantener las tres funciones juntas en un
solo `stock.py` (mismo nombre que el fichero legado) también conserva el mapeo 1:1 "de dónde vino
cada función", igual que la spec 014 preservó el agrupamiento `pricing.py`/`consumption.py`.

**Alternatives considered**:
- Replicar la separación núcleo/adaptador de `catalog_engine/` aunque no haya función pura que
  aislar: rechazado — sería una abstracción sin ningún requisito que la exija (regla de "no
  diseñar para lo hipotético" del proyecto); un `core.py` vacío o con un único helper trivial no
  aporta nada verificable que `stock.py` no aporte ya.
- Un fichero por función (`lock_items.py`, `record_movement.py`, `apply_adjustment.py`):
  rechazado — sobre-fragmentación no solicitada por ningún requisito, y las tres funciones son
  cortas (121 líneas en total hoy) y ya conviven en un solo fichero legado sin que eso sea un
  problema documentado.

## Decisión 3 — Mecánica de la fachada (Historia 3)

**Decision**: `app/api/v1/inventory/stock.py` queda reducido a imports explícitos desde
`app.inventory_engine` con reexport nombrado (`from app.inventory_engine import lock_items as
lock_items`, o equivalente vía `__all__`), sin ninguna función wrapper.

**Rationale**: FR-005 exige que la fachada "no contenga lógica propia de cálculo, validación o
consulta" — mismo criterio que FR-008 exigió en la spec 014, y la misma solución (reexport directo,
no wrapper) aplica sin cambios: un `import ... as ...` por símbolo es indistinguible en runtime de
que la función esté definida ahí mismo, que es justo lo que FR-006/SC-004 piden verificar (diff
vacío en los ficheros consumidores que no cambian).

**Alternatives considered**:
- Fachada por `from app.inventory_engine import *`: rechazado — misma razón que en la spec 014, no
  da el control fino que FR-001 exige (nombre exacto, misma firma) sin `__all__` explícito, que ya
  es más trabajo que tres imports nombrados.
- Funciones wrapper (`def lock_items(...): return inventory_engine.lock_items(...)`): rechazado —
  FR-005 prohíbe explícitamente "lógica propia"; un wrapper con firma repetida es exactamente ese
  tipo de duplicación evitable.

## Decisión 4 — FR-009: golden master nuevo vs. revisión manual documentada

**Decision**: opción (b) — revisión manual explícita y documentada de los tres sub-hallazgos de
A-35 en alcance, **sin** construir un golden master nuevo para inventario.

**Rationale**: la spec deja la elección abierta explícitamente para esta fase, y las dos opciones
son razonables en abstracto — pero comparadas contra lo que un golden master aporta *en este caso
concreto*, construir uno no se justifica:

1. **Los tres sub-hallazgos en alcance ya tienen test dedicado 1:1**, verificado leyendo
   `test_inventory_stock.py`: `test_rn_inv_05_allow_negative_true_permite_dejar_stock_negativo`
   cubre el sub-hallazgo 1 (`allow_negative=True`), `test_rn_inv_11_motivo_no_es_obligatorio` cubre
   el sub-hallazgo 2 (`reason` opcional), y
   `test_rn_inv_10_delta_cero_lanza_valueerror_no_http_exception` cubre el sub-hallazgo 3
   (`ValueError` sin handler) — con un comentario en el propio test que ya documenta la intención
   de congelar el tipo exacto de excepción. Un golden master existe para capturar comportamiento
   que characterization tests individuales *no* alcanzan a cubrir por su naturaleza encadenada; aquí
   no hay ningún hueco de cobertura que cerrar.
2. **No hay interacción encadenada entre funciones que justifique un golden master.** El motor de
   catálogo (spec 014) necesitó uno porque `compute_line_price`, `plan_line_consumption` y
   `validate_option_selection` interactúan en secuencias de varios pasos (variante → opciones →
   receta → descuento de inventario) donde un test aislado por función puede pasar sin que la
   *cadena completa* sea correcta — de ahí el valor de un escenario encadenado de 8-12 casos. Las
   tres funciones de `stock.py` son invocaciones independientes sobre una sola fila
   (`InventoryItem`): no hay una secuencia de negocio de varios pasos que una sola llamada a
   `record_movement` o `apply_adjustment` no capture ya.
3. **La batería comparativa de la Historia 2 (100-200 casos, FR-010) ya da cobertura más amplia
   que un golden master de 8-12 casos manuales** para esta superficie: barre combinaciones de tipo
   de movimiento, cantidades, `allow_negative`, y niveles de `current_stock` en y cerca de cero —
   exactamente el espacio donde vivirían los sub-hallazgos de A-35 — de forma generada y
   reproducible, no de un conjunto fijo escrito a mano.
4. Construir un golden master nuevo (módulo `golden_master_core.py` equivalente, fichero JSON de
   casos encadenados, test dedicado) es trabajo adicional no trivial que la Constitución no exige
   por sí sola — el Principio II pide que los characterization tests sean el árbitro, no que exista
   necesariamente un golden master por módulo; la spec 014 lo tuvo porque el módulo lo necesitaba,
   no porque sea un requisito universal.

**Revisión manual documentada (artefacto que satisface FR-009 opción b y SC-002)**:

| Sub-hallazgo A-35 | Test que lo caracteriza | Comportamiento verificado | Conclusión de la revisión |
|---|---|---|---|
| 1. `allow_negative=True` sin llamador visible | `test_rn_inv_05_allow_negative_true_permite_dejar_stock_negativo` (`test_inventory_stock.py`) | Con `allow_negative=True`, `record_movement` permite que `current_stock` quede negativo sin lanzar `InsufficientStockError`. Verificado además con `grep -rn "allow_negative=True" app/ --include="*.py"` (excluyendo characterization tests): cero resultados — confirma que ningún llamador de producción lo usa hoy. | El parámetro y su comportamiento se preservan tal cual en `app/inventory_engine/stock.py`; no se elimina ni se marca deprecado (edge case de `spec.md`). |
| 2. `reason` no obligatorio en `apply_adjustment` | `test_rn_inv_11_motivo_no_es_obligatorio` | Llamar `apply_adjustment` sin `reason` no lanza ningún error; el `InventoryMovement` resultante queda con `reason=None`. | Se preserva: `reason: Optional[str] = None` sin validación adicional en la firma movida. |
| 3. `signed_delta=0` propaga `ValueError` sin handler dedicado | `test_rn_inv_10_delta_cero_lanza_valueerror_no_http_exception` | `apply_adjustment(signed_delta=Decimal("0"))` lanza `ValueError("signed_delta must be != 0")`, no `InsufficientStockError` ni ninguna subclase de `HTTPException` — el test lo verifica explícitamente con `assertNotIsInstance(ctx.exception, InsufficientStockError)`. Sin handler en `app/main.py` para `ValueError` desnudo (no verificado de nuevo aquí — ya lo confirmó el registro de anomalías; fuera de esta spec corregirlo). | Se preserva sin capturar ni envolver — FR-003 lo exige explícitamente; el handler queda pendiente de spec futura dedicada, según el propio registro de anomalías. |

**Conclusión**: los tres sub-hallazgos quedan verificados por revisión de código + test existente,
sin discrepancias. La Historia 2 se da por cumplida en cuanto a FR-009 con este artefacto, sin
necesidad de construir un golden master de inventario.

**Alternatives considered**:
- Construir un golden master nuevo (opción a), mismo patrón que
  `golden_master_core.py`/`pricing_consumption.master.json`: rechazado por las cuatro razones de
  arriba — el costo de construirlo no compra cobertura adicional real sobre lo que ya dan los 16
  characterization tests más la batería de 100-200 casos de la Historia 2.

## Decisión 5 — Herramienta para el generador determinista de la Historia 2

**Decision**: `random.Random(seed)` de la biblioteca estándar de Python, mismo patrón que
`catalog_engine_equivalence_gate.py` de la spec 014.

**Rationale**: la Constitución (Principio IV) prefiere explícitamente la biblioteca estándar
cuando resuelve el mismo problema. La clarificación de sesión 2026-08-17 en `spec.md` ya fija el
tamaño (100-200 casos) y la reutilización de las factorías existentes de `fixtures.py`
(`f.make_inventory_item`, `f.new_session`) sin fixture nuevo — un espacio de combinaciones acotado
(tipo de movimiento, cantidad, `allow_negative`, `current_stock` inicial cerca de y en cero) que
`random.Random(seed).choice(...)`/`.sample(...)` cubre sin ningún paquete adicional.

**Alternatives considered**:
- `hypothesis` (property-based testing): rechazado — mismo razonamiento que la spec 014: el
  espacio no es infinito ni requiere shrinking, es una combinatoria acotada sobre datos ya
  conocidos del fixture.
- `random` global con `random.seed(seed)`: rechazado — muta estado global de proceso, con riesgo
  de fuga entre tests que corren en el mismo proceso `unittest`. Una instancia `Random(seed)`
  propia es autocontenida.

## Decisión 6 — Convención de ejecución de tests

**Decision**: la Historia 2 se ejecuta con `python3 -m unittest`, igual que el resto de
`app/characterization_tests/`.

**Rationale**: se verificó (no hay `pytest.ini` ni configuración de `pytest` en `pos-backend`) que
todos los characterization tests existentes, incluidos `test_inventory_stock.py` y
`catalog_engine_equivalence_gate.py` de la spec 014, usan `unittest.TestCase`. Introducir `pytest`
solo para este nuevo test rompería esa uniformidad sin ningún beneficio que la spec pida, y
contaría como dependencia nueva a justificar bajo el Principio IV sin necesidad real.

**Alternatives considered**: `pytest` — rechazado por la razón de arriba; la parametrización de
100-200 casos generados es perfectamente expresable con `self.subTest(caso=caso)` dentro de un
único `test_...` de `unittest`, mismo patrón que ya usa `catalog_engine_equivalence_gate.py`.

## Resumen — incógnitas resueltas

No queda ningún `NEEDS CLARIFICATION` pendiente en el Technical Context de `plan.md`: lenguaje,
dependencias, storage, testing, plataforma, tipo de proyecto, y alcance ya estaban determinados por
el código existente y la clarificación de `spec.md`. Esta fase añadió las seis decisiones de
arriba, incluyendo la resolución explícita de FR-009 (Decisión 4), sin necesitar investigación
externa adicional.
