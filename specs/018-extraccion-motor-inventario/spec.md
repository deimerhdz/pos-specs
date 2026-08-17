# Feature Specification: Extracción del motor de stock de inventario (`inventory/stock.py`) a módulo independiente

**Feature Branch**: `018-extraccion-motor-inventario`

**Created**: 2026-08-17

**Status**: Draft

**Input**: User description: "Extracción y modernización de inventory/stock.py de pos-backend (app/api/v1/inventory/stock.py) según el patrón strangler fig. Alcance reducido a stock.py — NO incluye inventory/service.py..."

**Naturaleza de esta spec**: **extracción de módulo por estrangulamiento** (Principio III de la
[Constitución](../../.specify/memory/constitution.md)), no una spec de característica nueva ni de
ingeniería inversa — mismo patrón ya aplicado en
[014-extraccion-motor-catalogo](../014-extraccion-motor-catalogo/spec.md). El comportamiento a
preservar ya está caracterizado por trabajo previo: los 16 characterization tests de
`app/characterization_tests/test_inventory_stock.py`¹, y las entradas A-35 (tres de sus cuatro
sub-hallazgos) y A-17 de
[`registro-de-anomalias.md`](../000-reconocimiento/registro-de-anomalias.md), en la parte que toca
a `inventory/stock.py`. Esta spec no redefine ni una sola regla de negocio: define **cómo mover el
código sin romper ese contrato**.

¹ Verificado contra el repositorio en el momento de escribir esta spec: 16 tests `def test_...` en
el fichero, repartidos en tres clases (`RecordMovementTests`: 7, `ApplyAdjustmentTests`: 6,
`LockItemsTests`: 3) — coincide con el número citado en el encargo.

## Contexto — qué existe hoy y qué protege esta extracción

### Inventario función por función de `app/api/v1/inventory/stock.py`

El fichero expone hoy exactamente tres funciones públicas, todas impuras (reciben `db: Session` y
mutan filas de `InventoryItem`/`InventoryMovement` bajo `SELECT ... FOR UPDATE`). No hay ninguna
función privada (`_prefijo`) ni ningún dataclass o tipo auxiliar propio del módulo.

| # | Función | Firma | Pureza | Qué hace |
|---|---|---|---|---|
| 1 | `lock_items` | `(db: Session, item_ids: Iterable[UUID]) -> dict[UUID, InventoryItem]` | Impura — SQL con `.with_for_update()` | Deduplica y ordena los IDs recibidos, bloquea en una sola consulta (orden canónico por `id`, para evitar deadlock entre transacciones que bloquean los mismos insumos en distinto orden) y devuelve un diccionario `{id: InventoryItem}`. Con lista vacía devuelve `{}` sin consultar. |
| 2 | `record_movement` | `(db: Session, inventory_item_id: UUID, *, type: str, quantity: Decimal, reason: Optional[str] = None, reference_type: Optional[str] = None, reference_id: Optional[UUID] = None, user_id: Optional[UUID] = None, allow_negative: bool = False) -> InventoryMovement` | Impura — SQL con `.with_for_update()`, muta `current_stock`, inserta `InventoryMovement` | Valida `quantity > 0` y `type in ("in", "out")` (rechaza `"adjustment"` explícitamente, ver A-35/RN-INV-?? más abajo). Bloquea la fila del insumo, calcula el delta con signo según `type`, rechaza el movimiento con `InsufficientStockError` si dejaría el stock negativo salvo `allow_negative=True`, aplica el nuevo stock y registra el movimiento en el kardex. |
| 3 | `apply_adjustment` | `(db: Session, inventory_item_id: UUID, *, signed_delta: Decimal, reason: Optional[str] = None, user_id: Optional[UUID] = None) -> InventoryMovement` | Impura — SQL con `.with_for_update()`, muta `current_stock`, inserta `InventoryMovement` | Valida `signed_delta != 0` (con `ValueError`, sin `allow_negative` — el ajuste manual nunca permite negativo). Bloquea la fila, calcula el nuevo stock sumando el delta con signo, rechaza si quedaría negativo, aplica el cambio y registra un movimiento `type="adjustment"` con `quantity=abs(signed_delta)` (la magnitud siempre positiva; el signo vive solo en `current_stock`). |

Las tres funciones comparten el mismo patrón `SELECT ... FOR UPDATE` — citado explícitamente en
**A-17** como la referencia que el resto del sistema (caja, reparto de mesa, apertura de sesión QR)
debería copiar y hoy no copia. Esta spec **no toca ese patrón**, solo lo traslada de ubicación.

Ningún import de `stock.py` depende de otro módulo de `app/api/v1/`; sus únicas dependencias son
`sqlalchemy`, `app.core.exceptions.InsufficientStockError`, `app.models.inventory_item.InventoryItem`
y `app.models.inventory_movement.InventoryMovement`.

### Consumidores actuales

Cuatro ficheros importan directamente de `app/api/v1/inventory/stock.py` hoy:

- `app/api/v1/inventory/router.py` — importa `apply_adjustment`.
- `app/api/v1/inventory/service.py` — importa `record_movement`. **Este fichero no cambia aquí**
  (ver "Fuera de alcance" abajo); solo se preserva su import.
- `app/api/v1/sales/consumption.py` — importa `lock_items` y `record_movement`.
- `app/api/v1/orders/consumption.py` — importa `lock_items` y `record_movement`.

Ninguno de los cuatro cambia en esta spec.

### Comportamiento protegido — contrato inmutable

Los 16 characterization tests de `test_inventory_stock.py` son la referencia primaria. Además, dos
entradas del registro de anomalías documentan comportamiento específico de `stock.py` que se
reproduce tal cual, sin corregir:

- **A-17** (citada arriba): el patrón `SELECT ... FOR UPDATE` de `stock.py` es la referencia
  positiva a preservar exactamente — no es un defecto, es el ejemplo a imitar en otros módulos
  (fuera de alcance aquí).
- **A-35** — cluster de cuatro sub-hallazgos del módulo `inventory`, de los cuales **tres** tocan
  código dentro de `stock.py` y uno vive en `inventory/service.py` (fuera de alcance de esta spec):
  1. `allow_negative=True` en `record_movement` (línea 49/72 del fichero actual) no tiene ningún
     llamador visible hoy — verificado: ningún fichero del repositorio invoca
     `record_movement(..., allow_negative=True)`. Se conserva el parámetro con su comportamiento
     actual tal cual, sin llamador, porque corregir o eliminar algo sin decisión de negocio viola
     el Principio I de la Constitución. **PENDIENTE** (sin decisión de negocio).
  2. El motivo (`reason`) de `apply_adjustment` no es obligatorio (líneas 97/117 del fichero
     actual) — el test `test_rn_inv_11_motivo_no_es_obligatorio` caracteriza esto explícitamente.
     **PENDIENTE** (sin decisión de negocio sobre si debería exigirse).
  3. Un ajuste con `signed_delta=0` se rechaza con `ValueError` (líneas 102-103 del fichero
     actual), que al no tener handler dedicado en `app/main.py` propaga como 500 genérico en vez de
     400/422 — caracterizado explícitamente por
     `test_rn_inv_10_delta_cero_lanza_valueerror_no_http_exception`, con comentario en el propio
     test que documenta la intención de congelar el tipo exacto de excepción. **ACCIDENTAL**
     (confirmado, el propio registro señala que esto se corrige "de inmediato" en una spec futura
     dedicada al handler — no aquí).
  4. *(Fuera de alcance de esta spec)* — el costo unitario se sobrescribe siempre en cada
     compra/recepción sin promediar (RN-INV-17), pero ese código vive en
     `inventory/service.py:76,149-151`, no en `stock.py`. Se documenta aquí solo para dejar
     explícito por qué el "clúster de 4" de A-35 aporta únicamente 3 sub-hallazgos al contrato de
     esta spec.

**A-13** (mencionada en el encargo original) se revisó contra el registro de anomalías: sus tres
referencias de código (`inventory/service.py:24-43`, `inventory/router.py:51-61`,
`reports/service.py:114-128`) no tocan `stock.py` en ningún punto — es un hallazgo sobre el
criterio de "bajo mínimo", no sobre las mutaciones de stock. **No aporta contrato a esta spec**; se
deja fuera del alcance funcional (ver Assumptions).

## Clarifications

### Session 2026-08-17

- Q: ¿Cómo debe dimensionarse y generarse la "batería masiva" de casos deterministas de la
  Historia 2 (FR-010)? → A: Muestreo aleatorio con semilla fija (`random.Random(seed)`) que
  produzca entre 100 y 200 casos, reutilizando las factorías de fixtures existentes sin crear un
  fixture nuevo — mismo patrón que `catalog_engine_equivalence_gate.py` de la spec 014.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Extraer el motor de stock a `app/inventory_engine/` sin alterar su salida (Priority: P1)

Como responsable de la modernización, muevo las tres funciones (`lock_items`, `record_movement`,
`apply_adjustment`) de `app/api/v1/inventory/stock.py` a `app/inventory_engine/`, conservando
intacto el patrón `SELECT ... FOR UPDATE` que ya usan, de modo que el nuevo módulo produzca
exactamente la misma salida que el código legado ante la misma entrada — sin que ningún fichero
consumidor tenga que cambiar una sola línea.

**Why this priority**: es el entregable central de la spec. Sin una extracción verificablemente
equivalente no hay nada que conmutar, y las historias 2 y 3 no tienen sentido sin ella.

**Independent Test**: se puede verificar de forma aislada ejecutando los 16 characterization tests
de `test_inventory_stock.py` apuntando sus imports a `app/inventory_engine/` en lugar de
`app/api/v1/inventory/stock.py`, sin tocar ningún fichero consumidor.

**Acceptance Scenarios**:

1. **Given** los 16 characterization tests de `test_inventory_stock.py` pasan hoy contra el código
   legado, **When** esos mismos tests se ejecutan importando desde `app/inventory_engine/` en vez de
   `app/api/v1/inventory/stock.py`, **Then** los 16 pasan en verde sin modificar ni una aserción.
2. **Given** las tres funciones en su nueva ubicación, **When** se inspecciona el cuerpo de cada
   una, **Then** el `SELECT ... FOR UPDATE` (`.with_for_update()`) sigue presente exactamente en
   los mismos puntos, sin reordenarse ni envolverse en una abstracción nueva.
3. **Given** las tres funciones en su nueva ubicación, **When** se inspecciona su firma, **Then**
   nombre, parámetros (posicionales, keyword-only, valores por defecto), tipos y valor de retorno
   son idénticos a los del fichero legado.
4. **Given** una llamada a `record_movement` con `type="adjustment"`, **When** se ejecuta contra
   `app/inventory_engine/`, **Then** se rechaza con el mismo `ValueError` y mensaje que hoy (A-35
   sub-hallazgo, no se corrige).
5. **Given** un ajuste con `signed_delta=0`, **When** se ejecuta `apply_adjustment` contra
   `app/inventory_engine/`, **Then** propaga el mismo `ValueError` sin capturar (no se agrega
   manejo nuevo; A-35 sub-hallazgo 3, no se corrige aquí).

---

### User Story 2 - Verificación de equivalencia comparativa masiva (Priority: P2)

Como responsable de la modernización, además de los characterization tests existentes, decido y
ejecuto una segunda red de verificación independiente para inventario — dado que, a diferencia del
motor de catálogo, no existe hoy un golden master dedicado a este módulo — y adicionalmente
ejecuto una batería masiva y determinista de casos generados (semilla fija) contra la
implementación legada y contra `app/inventory_engine/` sobre el mismo estado de datos, exigiendo
igualdad exacta campo a campo entre ambas salidas.

**Why this priority**: los characterization tests cubren los casos que alguien ya pensó en
escribir; la ausencia de golden master en inventario deja un hueco de verificación que el motor de
catálogo sí tenía cubierto, y debe resolverse explícitamente (construir uno o sustituirlo por
revisión manual) antes de dar este anillo por cumplido; la batería generada cubre además
combinaciones que nadie anticipó.

**Independent Test**: se puede ejecutar de forma aislada como su propio test/script, sin depender
de que las historias 1 o 3 estén terminadas — aunque en la práctica solo tiene sentido correrla una
vez que `app/inventory_engine/` existe. Es un gate de verificación **previo a la Historia 3**: una
vez la conmutación queda verificada, la comparación legado-vs-nuevo deja de tener sentido (ambos
apuntan al mismo código) y el test/script se retira o archiva, documentando su resultado.

**Acceptance Scenarios**:

1. **Given** que no existe golden master dedicado a inventario, **When** se planifica esta
   historia, **Then** se documenta explícitamente una de las dos decisiones antes de continuar:
   (a) construir un golden master de inventario nuevo (mismo patrón que
   `golden_master_core.py`/`pricing_consumption.master.json` del motor de catálogo), o (b)
   sustituirlo por una revisión manual explícita y documentada de los cuatro sub-hallazgos citados
   en A-35 (los tres que tocan `stock.py`), dejando constancia escrita de esa revisión como
   artefacto de esta spec.
2. **Given** una semilla fija y un generador determinista de casos (combinaciones de tipo de
   movimiento, cantidades, `allow_negative`, insumos con distinto `current_stock` incluyendo
   límites exactos en cero), **When** se genera la batería de casos y se ejecutan ambas
   implementaciones sobre el mismo estado de datos/fixture, **Then** cada corrida con la misma
   semilla produce exactamente los mismos casos (reproducibilidad del generador en sí, verificada
   antes de comparar implementaciones).
3. **Given** la batería generada, **When** se compara campo a campo la salida de la implementación
   legada contra `app/inventory_engine/` para cada caso (incluyendo el `current_stock` resultante
   del insumo, los campos del `InventoryMovement` creado, y el tipo/mensaje de cualquier excepción
   levantada), **Then** no hay ninguna diferencia — incluyendo los casos que ejercitan los tres
   sub-hallazgos de A-35 en alcance.
4. **Given** que la batería encuentra una sola diferencia, **When** se reporta el resultado,
   **Then** el reporte identifica el caso exacto (entrada + campo que difiere + valor legado vs.
   valor nuevo) para que sea reproducible sin tener que re-ejecutar toda la batería.

---

### User Story 3 - Conmutación final a fachada (Priority: P3)

Como responsable de la modernización, una vez que las historias 1 y 2 están en verde,
convierto `app/api/v1/inventory/stock.py` en una fachada pura que reexporta desde
`app/inventory_engine/`, de modo que los cuatro ficheros consumidores sigan importando exactamente
de la misma ruta que hoy, sin ningún cambio en su código.

**Why this priority**: es el paso que hace la extracción "real" para el resto del sistema — hasta
este punto, `app/inventory_engine/` podría existir en paralelo sin que nada dependa de él todavía.
Es la de menor prioridad de las tres porque, sin las historias 1 y 2 ya verificadas en verde, no
hay base para hacerla con confianza.

**Independent Test**: se puede verificar corriendo la suite completa de tests del backend (no solo
los de inventario) después de la conmutación, sin haber tocado ningún fichero consumidor.

**Acceptance Scenarios**:

1. **Given** `app/api/v1/inventory/stock.py` convertido en fachada, **When** se inspecciona su
   contenido, **Then** las tres funciones que exportaba antes (`lock_items`, `record_movement`,
   `apply_adjustment`) siguen siendo importables desde la misma ruta y el mismo nombre de módulo,
   y el fichero no contiene lógica propia de cálculo, validación o consulta.
2. **Given** los cuatro ficheros consumidores (`inventory/router.py`, `inventory/service.py`,
   `sales/consumption.py`, `orders/consumption.py`), **When** se compara su contenido antes y
   después de la conmutación, **Then** no hay ninguna diferencia — ni en sus imports ni en el resto
   de su código.
3. **Given** la suite completa de tests del backend (no solo characterization tests de inventario),
   **When** se ejecuta después de la conmutación, **Then** pasa exactamente igual que antes de
   empezar la extracción (mismos tests en verde, mismos tests en rojo si los había).

### Edge Cases

- ¿Qué pasa si la batería de casos generados (Historia 2) encuentra una diferencia que **no** está
  cubierta por ningún characterization test existente? → Bloquea la extracción: la diferencia se
  investiga y, o es un error de la extracción (se corrige antes de continuar), o revela un caso
  real no caracterizado (se documenta como anomalía nueva en `registro-de-anomalias.md` antes de
  decidir cómo tratarlo — nunca se ignora ni se ajusta la batería para que deje de detectarlo).
- ¿Qué pasa con el `allow_negative=True` sin llamador visible (A-35 sub-hallazgo 1) durante la
  extracción? → Se conserva el parámetro y su comportamiento actual tal cual, incluida su
  característica de "código muerto en la práctica" — no se elimina ni se documenta como deprecado,
  porque eso sería un cambio de comportamiento/superficie sin decisión de negocio.
- ¿Qué pasa si, durante la extracción, aparece un quinto consumidor de `stock.py` no identificado
  en esta spec? → Se documenta y se incluye en el alcance de "ficheros consumidores que no cambian"
  antes de continuar con la Historia 3; no invalida las Historias 1 y 2.
- ¿Qué pasa si `inventory/service.py` necesita cambiar como efecto colateral de esta extracción
  (por ejemplo, para ajustar su import)? → Solo se permite el cambio mínimo de import necesario
  para que siga apuntando a `record_movement` en su ubicación correcta (directamente o vía la
  fachada); ningún otro cambio a `service.py` es parte de esta spec — su propia extracción requiere
  spec de prerrequisito aparte, dado que no tiene cobertura de characterization tests confirmada.
- ¿Qué pasa si se decide no construir un golden master de inventario (Historia 2, opción b)? →
  La revisión manual explícita de los tres sub-hallazgos de A-35 en alcance debe quedar documentada
  como artefacto verificable de esta spec (por ejemplo, en `quickstart.md` o `research.md` de la
  fase de planificación), no como una nota informal; sin ese artefacto, la Historia 2 no se
  considera completa.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: El sistema DEBE exponer, en `app/inventory_engine/`, las tres funciones hoy en
  `app/api/v1/inventory/stock.py` (`lock_items`, `record_movement`, `apply_adjustment`), con la
  misma firma (nombre, parámetros incluyendo keyword-only y valores por defecto, tipos, valor de
  retorno) que tienen hoy.
- **FR-002**: El sistema DEBE conservar, en cada una de las tres funciones movidas, el patrón
  `SELECT ... FOR UPDATE` (`.with_for_update()`) exactamente en los mismos puntos del flujo que
  tiene hoy, sin alterar el orden de bloqueo ni introducir una abstracción nueva sobre él — este
  patrón es la referencia citada en A-17 que otros módulos deberían copiar.
- **FR-003**: El sistema DEBE reproducir en `app/inventory_engine/`, sin corregirlos, los tres
  sub-hallazgos de A-35 que tocan `stock.py`: (1) `allow_negative=True` sin llamador visible hoy en
  el repositorio, conservado tal cual; (2) el motivo (`reason`) de `apply_adjustment` sigue sin ser
  obligatorio; (3) un ajuste con `signed_delta=0` sigue propagando `ValueError` sin handler
  dedicado (500 genérico en la app real), sin capturarlo ni convertirlo a una respuesta controlada.
- **FR-004**: El sistema NO DEBE corregir, mitigar ni alterar de ningún modo el sub-hallazgo 4 de
  A-35 (costo unitario siempre sobrescrito sin promediar) ni ninguna otra anomalía de
  `inventory/service.py`, `inventory/router.py` o `reports/service.py` (incluyendo A-13) como parte
  de esta extracción — esos ficheros quedan fuera de todo cambio salvo el ajuste mínimo de import
  descrito en FR-008.
- **FR-005**: `app/api/v1/inventory/stock.py`, al concluir la conmutación (Historia 3), DEBE quedar
  como fachada que reexporta exclusivamente desde `app/inventory_engine/`, sin contener lógica
  propia de cálculo, validación o consulta.
- **FR-006**: Ninguno de los cuatro ficheros consumidores (`inventory/router.py`,
  `sales/consumption.py`, `orders/consumption.py`) DEBE modificarse como parte de esta spec, salvo
  `inventory/service.py` en el alcance mínimo de FR-008.
- **FR-007**: El sistema DEBE pasar, sin modificar ninguna aserción existente, los 16
  characterization tests de `test_inventory_stock.py` ejecutados contra `app/inventory_engine/`.
- **FR-008**: `inventory/service.py` DEBE seguir importando `record_movement` exitosamente después
  de la conmutación (directamente desde `app/inventory_engine/` o vía la fachada de
  `stock.py`), sin que su comportamiento observable cambie; cualquier ajuste de import en este
  fichero es el único cambio permitido en él, y no constituye su extracción — esa requiere su
  propia spec de prerrequisito, fuera de alcance aquí.
- **FR-009**: El sistema DEBE decidir explícitamente, antes de dar por cumplida la Historia 2, entre
  (a) construir un golden master dedicado a inventario (mismo patrón que el ya existente para el
  motor de catálogo) o (b) sustituirlo por una revisión manual explícita y documentada de los tres
  sub-hallazgos de A-35 en alcance — no puede quedar sin decidir ni omitirse en silencio.
- **FR-010**: El sistema DEBE incluir un test de equivalencia comparativa que, usando
  `random.Random(semilla_fija)`, genere entre 100 y 200 casos deterministas (movimientos
  `in`/`out`/rechazados, ajustes con signo positivo/negativo/cero, `allow_negative` en ambos
  valores, insumos con `current_stock` en y cerca de cero) a partir de las mismas factorías de
  fixtures ya usadas por los characterization tests (sin crear un fixture nuevo) — mismo patrón que
  `app/characterization_tests/catalog_engine_equivalence_gate.py` de la spec 014 —, ejecute la
  implementación legada y `app/inventory_engine/` sobre ese mismo estado de datos para cada caso, y
  falle si algún campo de la salida (incluyendo tipo y mensaje de excepción) difiere entre ambas.
  Este test es un gate de verificación previo a la conmutación (Historia 3): una vez pasa en verde y
  la conmutación queda verificada, deja de tener sentido y se retira o archiva, documentando su
  resultado — no forma parte de la suite de regresión permanente, que queda cubierta por los
  characterization tests (FR-007) y, si se construye, el golden master (FR-009 opción a).
- **FR-011**: Cualquier diferencia de comportamiento detectada por cualquiera de los anillos de
  verificación (characterization tests, golden master si se construye, revisión manual si se
  elige en su lugar, o la batería comparativa) DEBE tratarse como una regresión que bloquea la
  conmutación (Historia 3) hasta resolverse — nunca ajustando el test o el criterio para que la
  diferencia deje de detectarse, salvo que ya exista una decisión de negocio registrada en
  `registro-de-anomalias.md` que la autorice explícitamente (Principio I de la Constitución).

### Key Entities *(include if feature involves data)*

- **Motor de stock**: las tres funciones (`lock_items`, `record_movement`, `apply_adjustment`) que
  mutan `current_stock` de un `InventoryItem` y registran el movimiento correspondiente en
  `InventoryMovement`, siempre bajo bloqueo de fila (`SELECT ... FOR UPDATE`).
- **Contrato de comportamiento**: el conjunto formado por los 16 characterization tests de
  `test_inventory_stock.py` y los tres sub-hallazgos de A-35 en alcance (más A-17 como referencia
  positiva del patrón de bloqueo) — la referencia única frente a la que se mide si la extracción es
  equivalente.
- **Batería comparativa**: el conjunto de casos generados de forma determinista (semilla fija) para
  la Historia 2, junto con el estado de datos/fixture fijo sobre el que se ejecutan ambas
  implementaciones.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: El 100% de los 16 characterization tests de `test_inventory_stock.py` pasan
  ejecutados contra `app/inventory_engine/`, sin ninguna aserción modificada respecto a su versión
  contra el código legado.
- **SC-002**: La decisión entre golden master nuevo o revisión manual explícita (FR-009) queda
  documentada por escrito antes de dar la Historia 2 por cumplida, con evidencia verificable
  (fichero de golden master o documento de revisión).
- **SC-003**: La batería de equivalencia comparativa (semilla fija) reporta cero diferencias campo
  a campo entre la implementación legada y `app/inventory_engine/` en el 100% de los casos
  generados.
- **SC-004**: Los cuatro ficheros consumidores tienen cero líneas modificadas (diff vacío) al
  concluir la conmutación, salvo el ajuste mínimo de import permitido en `inventory/service.py`
  (FR-008), comparados contra su estado inmediatamente anterior a esta spec.
- **SC-005**: La suite completa de tests del backend (no solo los de inventario) pasa exactamente
  igual después de la conmutación que antes de empezar la extracción — mismo conjunto de tests en
  verde.
- **SC-006**: El patrón `SELECT ... FOR UPDATE` aparece en `app/inventory_engine/` el mismo número
  de veces y en los mismos puntos lógicos (uno por función) que en el fichero legado, verificable
  por inspección estática.

## Assumptions

- `app/inventory_engine/` es la ruta de destino, siguiendo la misma convención de nombrado que
  `app/catalog_engine/` (ya extraído en la spec 014, junto a `app/models/`, `app/core/`); si la
  convención real del proyecto al ejecutar esta spec indica otra ruta, se ajusta manteniendo el
  nombre del paquete como referencia estable para los imports internos que se creen durante la
  extracción.
- A-13 (mencionada en el encargo original) no aporta contrato a esta spec: sus tres referencias de
  código en el registro de anomalías (`inventory/service.py`, `inventory/router.py`,
  `reports/service.py`) no tocan `stock.py`. Se documenta esta verificación en el Contexto para que
  quede explícito por qué no genera ningún requisito funcional aquí.
- Del "clúster de 4 sub-hallazgos" de A-35 citado en el encargo, solo tres tocan código dentro de
  `stock.py` (`allow_negative` sin llamador, motivo no obligatorio, `signed_delta=0` sin handler);
  el cuarto (costo unitario siempre sobrescrito, RN-INV-17) vive en `inventory/service.py` y queda
  fuera de alcance funcional, aunque se documenta en el Contexto para trazabilidad completa del
  clúster.
- `inventory/service.py` no se extrae en esta spec por no tener cobertura de characterization tests
  confirmada en el análisis base (según lo indicado en el encargo); el único cambio permitido en
  ese fichero es el ajuste mínimo de import necesario para que siga funcionando tras la conmutación
  (FR-008), nunca la extracción de su lógica propia.
- La batería comparativa de la Historia 2 corre en un entorno de test aislado (misma base de datos
  de fixture para ambas corridas, sin efectos secundarios sobre datos de producción), igual que
  corren hoy los characterization tests.
- El criterio de "igualdad exacta campo a campo" en la Historia 2 incluye tanto el `current_stock`
  resultante y los campos del `InventoryMovement` creado, como el tipo y contenido de cualquier
  excepción levantada (por ejemplo, `InsufficientStockError` con el mismo mensaje, o `ValueError`
  con el mismo texto) — una diferencia en el mensaje de error cuenta como diferencia, no solo una
  diferencia en el valor calculado.
- La decisión entre construir un golden master de inventario o sustituirlo por revisión manual
  (FR-009) se deja como decisión de planificación (`/speckit-plan`), no se fuerza aquí una opción
  por defecto, porque ambas son razonables y el encargo original pide explícitamente decidir en esa
  fase antes de dar el anillo de verificación por cumplido.
