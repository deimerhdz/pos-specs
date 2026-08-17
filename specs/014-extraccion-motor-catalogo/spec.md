# Feature Specification: Extracción del motor de catálogo (`line_pricing.py` + `consumption_plan.py`) a módulo independiente

**Feature Branch**: `014-extraccion-motor-catalogo`

**Created**: 2026-08-17

**Status**: Draft

**Naturaleza de esta spec**: **extracción de módulo por estrangulamiento** (Principio III de la
[Constitución](../../.specify/memory/constitution.md)), no una spec de característica nueva ni de
ingeniería inversa. El comportamiento a preservar ya está caracterizado por trabajo previo: los 45
characterization tests de `app/characterization_tests/test_catalog_line_pricing.py` (25) y
`test_catalog_consumption_plan.py` (16)¹, el golden master (`golden_master_core.py` +
`test_golden_master_pricing_consumption.py` + `pricing_consumption.master.json`), y las entradas
A-02, A-03, A-05, A-06, A-32, A-33, A-36 (parcial) y A-47 (parcial) de
[`registro-de-anomalias.md`](../000-reconocimiento/registro-de-anomalias.md). Esta spec no
redefine ni una sola regla de negocio: define **cómo mover el código sin romper ese contrato**.

¹ Verificado contra el repositorio en el momento de escribir esta spec: 25 + 16 = 41 tests
`def test_...` en los dos ficheros. El número 45 con el que se abrió el encargo no coincide con el
conteo por función; se documenta la discrepancia en Supuestos y **no** bloquea la spec — el
criterio de aceptación es "todos los tests de ambos ficheros pasan", no un conteo fijo.

## Contexto — qué existe hoy y qué protege esta extracción

El motor de catálogo vive hoy en dos ficheros bajo `app/api/v1/catalog/`:

- **`line_pricing.py`**: `compute_line_price`, `load_valid_options`, `check_availability`,
  `validate_option_selection`, `grupos_que_descuentan`, `_exige_maximo`. Además reexporta desde
  `consumption_plan.py` (líneas 31-36) `ConsumptionLine`, `load_variant_groups`,
  `plan_line_consumption`, `required_consumption` — reexport del que depende
  `app/api/v1/cart/service.py:31-36`.
- **`consumption_plan.py`**: `load_recipe`, `load_variant_groups`, `plan_line_consumption`,
  `required_consumption`, `group_discounts`, `variant_label`, `ensure_lines_consume_inventory`.

De estas trece funciones, únicamente `compute_line_price` y `_exige_maximo` son hoy puras (no
reciben `db: Session` ni tocan SQLAlchemy). Las once restantes reciben `db: Session` y consultan o
tocan directamente los modelos `Option`, `OptionGroup`, `ProductVariant`, `InventoryItem`,
`Product`, `RecipeItem` y `VariantOptionGroup`.

Siete ficheros consumen este motor hoy: `sales/service.py`, `sales/consumption.py`,
`orders/service.py`, `orders/consolidation.py`, `orders/kitchen.py`, `orders/consumption.py` y
`cart/service.py`. Ninguno de los siete cambia en esta spec.

El comportamiento que este motor produce hoy incluye, entre otras, dos invariantes marcadas
`[PROTEGIDA]` en el registro de anomalías — comportamientos intencionales, verificados con dos
testigos independientes, que **no se corrigen aquí aunque parezcan defectos**:

- **A-02**: el consumo por opción usa una sola cantidad (la del tamaño manda sobre la de la
  opción; nunca se suman) — corrige un bug histórico de doble descuento de 140g, y la corrección
  misma es la invariante protegida.
- **A-05**: `STRICT_OPTION_SELECTION=False` por defecto es una tolerancia deliberada de migración
  del catálogo histórico, no un descuido.

Y varias anomalías documentadas como `PENDIENTE` o `ACCIDENTAL` que tampoco se corrigen aquí — se
reproducen tal cual porque corregirlas sin decisión de negocio violaría el Principio I de la
Constitución:

- **A-03**: el docstring de `VariantOptionGroup.quantity_per_option` sigue describiendo el
  comportamiento pre-A-02 ("se suman"), contradiciendo el código real.
- **A-06**: con la tolerancia de A-05 activa, se puede cobrar y consumir inventario de una opción
  de un grupo que la variante no ofrece.
- **A-32**: `grupos_que_descuentan` (en `line_pricing.py`) y `group_discounts` (en
  `consumption_plan.py`) usan criterios distintos para la misma pregunta — el primero no exige
  `active` ni `inventory_item_id`, el segundo sí.
- **A-33**: un grupo opcional (`min_select=0`) que es la única fuente de consumo de inventario de
  una variante bloquea la venta con 409 si nadie elige nada, aunque el propio código llama a "no
  elegir" una decisión legítima en ese mismo caso.
- **A-36 (parcial, solo el punto 1)**: no hay redondeo explícito en el precio de línea que
  devuelve `compute_line_price` — se conserva la suma decimal exacta tal cual.
- **A-47 (parcial)**: `check_availability` es un chequeo best-effort — no reserva ni bloquea
  stock; puede quedar obsoleto entre el armado del carrito y la confirmación del pedido.

Explícitamente **fuera** de esta extracción: **A-04** (falta `variant=variant` en
`orders/consolidation.py:199`). El defecto vive en un fichero consumidor, no en el motor que se
extrae aquí, y su corrección requiere su propia spec delta con decisión de negocio.

## Clarifications

### Session 2026-08-17

- Q: Una vez completada la Historia 3 (conmutación a fachada), ¿la batería comparativa de la
  Historia 2 (FR-013) debe quedar como test permanente en el repositorio, o era desde el inicio
  un gate de verificación único que corre antes de la conmutación y luego se retira? → A: Es un
  gate de verificación único, previo a la Historia 3. Tras la conmutación, `line_pricing.py` y
  `consumption_plan.py` son fachadas de `app/catalog_engine/`, así que comparar "legado" contra
  "nuevo" deja de tener sentido (ambos apuntan al mismo código). El test/script se retira o
  archiva una vez la conmutación queda verificada, documentando su resultado; los characterization
  tests y el golden master quedan como la red de regresión permanente.
- Q: ¿Qué orden de magnitud de casos debe generar la batería determinista de la Historia 2
  (FR-013) por corrida? → A: Entre 100 y 200 casos por corrida (semilla fija), suficiente para
  cubrir combinaciones de variantes/grupos de opciones/recetas/niveles de stock más allá de los 41
  characterization tests, sin que el gate previo a la Historia 3 se vuelva pesado de ejecutar.
- Q: ¿La batería comparativa (Historia 2) reutiliza el fixture/base de datos ya usado por los
  characterization tests y el golden master, o necesita su propio fixture dedicado? → A: Reutiliza
  el fixture existente de los characterization tests y el golden master; el generador de casos
  toma sus combinaciones de variantes/grupos de opciones/recetas/niveles de stock de ese mismo
  dataset ya sembrado, sin construir infraestructura de fixtures nueva.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Extraer el motor a `app/catalog_engine/` sin alterar su salida (Priority: P1)

Como responsable de la modernización, muevo las trece funciones y el dataclass
`ConsumptionLine` del motor de catálogo a `app/catalog_engine/`, separando el núcleo puro
(`compute_line_price`, `_exige_maximo`) de los adaptadores que hacen I/O, de modo que el nuevo
módulo produzca exactamente la misma salida que el código legado ante la misma entrada — sin que
ningún fichero consumidor tenga que cambiar una sola línea.

**Why this priority**: es el entregable central de la spec. Sin una extracción verificablemente
equivalente no hay nada que conmutar, y las historias 2 y 3 no tienen sentido sin ella.

**Independent Test**: se puede verificar de forma aislada ejecutando los 41 characterization
tests y el golden master (sin regenerar) apuntando sus imports a `app/catalog_engine/` en lugar de
a `app/api/v1/catalog/`, sin tocar ningún fichero consumidor.

**Acceptance Scenarios**:

1. **Given** los 41 characterization tests de `test_catalog_line_pricing.py` y
   `test_catalog_consumption_plan.py` pasan hoy contra el código legado, **When** esos mismos
   tests se ejecutan importando desde `app/catalog_engine/` en vez de
   `app/api/v1/catalog/`, **Then** los 41 pasan en verde sin modificar ni una aserción.
2. **Given** el golden master (`pricing_consumption.master.json`) ya generado contra el código
   legado, **When** `test_golden_master_pricing_consumption.py` se ejecuta apuntando a
   `app/catalog_engine/`, **Then** el master pasa **sin regenerarse**.
3. **Given** las funciones `compute_line_price` y `_exige_maximo` en su nueva ubicación, **When**
   se inspeccionan sus imports, **Then** ninguna importa `sqlalchemy` ni recibe un parámetro
   `db: Session`.
4. **Given** las once funciones adaptadoras en su nueva ubicación, **When** se inspecciona su
   firma, **Then** todas reciben `db: Session` en la misma posición que en el código legado.
5. **Given** una entrada que ejercita la invariante `[PROTEGIDA]` A-02 (tamaño con
   `quantity_per_option>0` y opción con `item_quantity>0` distinto), **When** se calcula el
   consumo con `app/catalog_engine/`, **Then** el resultado descuenta únicamente la cantidad del
   tamaño, igual que hoy — nunca la suma de ambas.

---

### User Story 2 - Verificación de equivalencia comparativa masiva (Priority: P2)

Como responsable de la modernización, además de los characterization tests y el golden master
existentes, ejecuto una batería masiva y determinista de casos generados (semilla fija) contra
la implementación legada y contra `app/catalog_engine/` sobre el mismo estado de datos, y exijo
igualdad exacta campo a campo entre ambas salidas, para tener una tercera señal de equivalencia
independiente de los tests ya escritos (que, por definición, no pueden cubrir cada combinación
posible de variante/opciones/grupos).

**Why this priority**: los characterization tests y el golden master cubren los casos que alguien
ya pensó en escribir; una batería generada con semilla fija cubre combinaciones que nadie
anticipó, y es la red de seguridad real contra una extracción que "pasa los tests pero cambia
algo sutil" en un caso no cubierto.

**Independent Test**: se puede ejecutar de forma aislada como su propio test/script, sin depender
de que las historias 1 o 3 estén terminadas — aunque en la práctica solo tiene sentido correrla
una vez que `app/catalog_engine/` existe. Es un gate de verificación **previo a la Historia 3**: una
vez la conmutación queda verificada, la comparación legado-vs-nuevo deja de tener sentido (ambos
apuntan al mismo código) y el test/script se retira o archiva, documentando su resultado.

**Acceptance Scenarios**:

1. **Given** una semilla fija y un generador determinista de casos (combinaciones de variantes,
   grupos de opciones, recetas y niveles de stock), **When** se genera la batería de casos y se
   ejecutan ambas implementaciones sobre el mismo estado de datos/fixture, **Then** cada corrida
   con la misma semilla produce exactamente los mismos casos (reproducibilidad del generador en
   sí, verificada antes de comparar implementaciones).
2. **Given** la batería generada, **When** se compara campo a campo la salida de la
   implementación legada contra `app/catalog_engine/` para cada caso, **Then** no hay ninguna
   diferencia — incluyendo los casos que ejercitan A-02, A-05, A-06, A-32 y A-33.
3. **Given** que la batería encuentra una sola diferencia, **When** se reporta el resultado,
   **Then** el reporte identifica el caso exacto (entrada + campo que difiere + valor legado vs.
   valor nuevo) para que sea reproducible sin tener que re-ejecutar toda la batería.

---

### User Story 3 - Conmutación final a fachada (Priority: P3)

Como responsable de la modernización, una vez que las historias 1 y 2 están en verde, convierto
`line_pricing.py` y `consumption_plan.py` en fachadas puras que reexportan desde
`app/catalog_engine/`, de modo que los siete ficheros consumidores sigan importando exactamente
de la misma ruta que hoy, sin ningún cambio en su código.

**Why this priority**: es el paso que hace la extracción "real" para el resto del sistema — hasta
este punto, `app/catalog_engine/` podría existir en paralelo sin que nada dependa de él todavía.
Es la de menor prioridad de las tres porque, sin las historias 1 y 2 ya verificadas en verde, no
hay base para hacerla con confianza.

**Independent Test**: se puede verificar corriendo la suite completa de tests del backend (no
solo los de catálogo) después de la conmutación, sin haber tocado ningún fichero consumidor.

**Acceptance Scenarios**:

1. **Given** `line_pricing.py` y `consumption_plan.py` convertidos en fachada, **When** se
   inspecciona su contenido, **Then** cada símbolo público que exportaban antes (las trece
   funciones, `ConsumptionLine`, y el reexport de `line_pricing.py:31-36`) sigue siendo
   importable desde la misma ruta y el mismo nombre de módulo.
2. **Given** los siete ficheros consumidores (`sales/service.py`, `sales/consumption.py`,
   `orders/service.py`, `orders/consolidation.py`, `orders/kitchen.py`, `orders/consumption.py`,
   `cart/service.py`), **When** se compara su contenido antes y después de la conmutación,
   **Then** no hay ninguna diferencia — ni en sus imports ni en el resto de su código.
3. **Given** la suite completa de tests del backend (no solo characterization tests de catálogo),
   **When** se ejecuta después de la conmutación, **Then** pasa exactamente igual que antes de
   empezar la extracción (mismos tests en verde, mismos tests en rojo si los había).

### Edge Cases

- ¿Qué pasa si la batería de casos generados (Historia 2) encuentra una diferencia que **no**
  está cubierta por ningún characterization test existente? → Bloquea la extracción: la
  diferencia se investiga y, o es un error de la extracción (se corrige antes de continuar), o
  revela un caso real no caracterizado (se documenta como anomalía nueva en
  `registro-de-anomalias.md` antes de decidir cómo tratarlo — nunca se ignora ni se ajusta la
  batería para que deje de detectarlo).
- ¿Qué pasa si, durante la separación núcleo/adaptador, alguna de las once funciones "adaptadoras"
  resulta tener lógica pura mezclada con I/O que valdría la pena separar más? → Fuera de alcance:
  esta spec preserva la frontera funcional actual (trece funciones con las mismas firmas); no
  autoriza descomponer ninguna función existente en piezas más pequeñas.
- ¿Qué pasa si el reexport de `line_pricing.py:31-36` deja de ser necesario porque
  `cart/service.py` se moderniza en una spec futura? → Fuera de alcance de esta spec; el reexport
  se conserva mientras `cart/service.py` (sin modificar aquí) siga dependiendo de él.
- ¿Qué pasa si aparece una anomalía nueva, no listada en el contrato de esta spec, mientras se
  hace la extracción? → Se documenta en `registro-de-anomalias.md` (Principio I) y se reproduce
  tal cual en `app/catalog_engine/`; no se corrige como parte de esta spec salvo que ya exista
  decisión de negocio.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: El sistema DEBE exponer, en `app/catalog_engine/`, las trece funciones hoy
  repartidas en `line_pricing.py` (`compute_line_price`, `load_valid_options`,
  `check_availability`, `validate_option_selection`, `grupos_que_descuentan`, `_exige_maximo`) y
  `consumption_plan.py` (`load_recipe`, `load_variant_groups`, `plan_line_consumption`,
  `required_consumption`, `group_discounts`, `variant_label`,
  `ensure_lines_consume_inventory`), con la misma firma (nombre, parámetros, tipos, valor de
  retorno) que tienen hoy.
- **FR-002**: El sistema DEBE exponer también el dataclass `ConsumptionLine` desde
  `app/catalog_engine/` con los mismos campos (`inventory_item_id`, `quantity`, `source`) y el
  mismo comportamiento (`frozen=True`) que tiene hoy.
- **FR-003**: Dentro de `app/catalog_engine/`, el sistema DEBE separar físicamente el núcleo puro
  (`compute_line_price`, `_exige_maximo`) de las once funciones adaptadoras que reciben
  `db: Session`, de modo que el código del núcleo puro no importe `sqlalchemy` ni ningún módulo
  que a su vez dependa de `sqlalchemy`.
- **FR-004**: Las once funciones adaptadoras DEBEN conservar su dependencia de `db: Session` y su
  interacción con los modelos `Option`, `OptionGroup`, `ProductVariant`, `InventoryItem`,
  `Product`, `RecipeItem` y `VariantOptionGroup`, sin alterar qué consultan ni cómo.
- **FR-005**: El sistema DEBE reproducir en `app/catalog_engine/`, sin corregirlas, las
  invariantes `[PROTEGIDA]` A-02 (el tamaño manda sobre la opción en el consumo; nunca se suman)
  y A-05 (`STRICT_OPTION_SELECTION=False` como tolerancia deliberada por defecto).
- **FR-006**: El sistema DEBE reproducir en `app/catalog_engine/`, sin corregirlas, las anomalías
  documentadas A-03 (docstring desactualizado, si se traslada literalmente), A-06 (tolerancia de
  opción de grupo ajeno con `STRICT_OPTION_SELECTION=False`), A-32 (criterio distinto entre
  `grupos_que_descuentan` y `group_discounts` sobre qué grupo "descuenta"), A-33 (bloqueo de venta
  cuando un grupo opcional es la única fuente de consumo y nadie elige nada), A-36 punto 1 (sin
  redondeo explícito en `compute_line_price`) y A-47 (chequeo de disponibilidad best-effort, sin
  reserva ni bloqueo de stock).
- **FR-007**: El sistema NO DEBE corregir, mitigar ni alterar de ningún modo el defecto A-04
  (`variant=variant` faltante en `orders/consolidation.py:199`) como parte de esta extracción —
  el fichero consumidor donde vive ese defecto queda fuera de todo cambio.
- **FR-008**: `line_pricing.py` y `consumption_plan.py`, al concluir la conmutación (Historia 3),
  DEBEN quedar como fachadas que reexportan exclusivamente desde `app/catalog_engine/`, sin
  contener lógica propia de cálculo, validación o consulta.
- **FR-009**: El reexport que hoy hace `line_pricing.py:31-36` (`ConsumptionLine`,
  `load_variant_groups`, `plan_line_consumption`, `required_consumption`) DEBE seguir funcionando
  exactamente igual después de la conmutación, para que `cart/service.py:31-36` no rompa su
  import.
- **FR-010**: Ninguno de los siete ficheros consumidores (`sales/service.py`,
  `sales/consumption.py`, `orders/service.py`, `orders/consolidation.py`, `orders/kitchen.py`,
  `orders/consumption.py`, `cart/service.py`) DEBE modificarse como parte de esta spec.
- **FR-011**: El sistema DEBE pasar, sin modificar ninguna aserción existente, los tests de
  `test_catalog_line_pricing.py` y `test_catalog_consumption_plan.py` ejecutados contra
  `app/catalog_engine/`.
- **FR-012**: El sistema DEBE pasar el golden master (`test_golden_master_pricing_consumption.py`
  contra `pricing_consumption.master.json`) ejecutado contra `app/catalog_engine/`, sin
  regenerar el master.
- **FR-013**: El sistema DEBE incluir un test de equivalencia comparativa que, con una semilla
  fija, genere una batería determinista de entre 100 y 200 casos a partir del mismo fixture ya
  usado por los characterization tests y el golden master (sin fixture dedicado nuevo), ejecute la
  implementación legada y `app/catalog_engine/` sobre ese mismo estado de datos para cada caso, y
  falle si algún campo de la salida difiere entre ambas. Este test es un gate de verificación
  previo a la conmutación (Historia 3): una vez esta pasa en verde y la conmutación queda verificada, deja de
  tener sentido (legado y nuevo pasan a ser el mismo código) y se retira o archiva, documentando su
  resultado — no forma parte de la suite de regresión permanente, que queda cubierta por los
  characterization tests y el golden master (FR-011, FR-012).
- **FR-014**: Cualquier diferencia de comportamiento detectada por cualquiera de los tres anillos
  de verificación (characterization tests, golden master, o la batería comparativa) DEBE tratarse
  como una regresión que bloquea la conmutación (Historia 3) hasta resolverse — nunca ajustando el
  test o el master para que la diferencia deje de detectarse, salvo que ya exista una decisión de
  negocio registrada en `registro-de-anomalias.md` que la autorice explícitamente (Principio II de
  la Constitución).

### Key Entities *(include if feature involves data)*

- **Núcleo puro del motor de catálogo**: las funciones `compute_line_price` y `_exige_maximo` —
  calculan un resultado a partir únicamente de sus parámetros de entrada, sin leer ni escribir
  estado externo.
- **Adaptadores de I/O del motor de catálogo**: las once funciones restantes — envuelven una
  consulta o validación contra la base de datos (a través de `db: Session`) y delegan el cálculo
  puro, cuando aplica, en el núcleo.
- **Contrato de comportamiento**: el conjunto formado por los 41 characterization tests, el
  golden master, y las entradas del registro de anomalías citadas en esta spec — la referencia
  única frente a la que se mide si la extracción es equivalente.
- **Batería comparativa**: el conjunto de casos generados de forma determinista (semilla fija)
  para la Historia 2, junto con el estado de datos/fixture fijo sobre el que se ejecutan ambas
  implementaciones.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: El 100% de los characterization tests de `test_catalog_line_pricing.py` (25) y
  `test_catalog_consumption_plan.py` (16) pasan ejecutados contra `app/catalog_engine/`, sin
  ninguna aserción modificada respecto a su versión contra el código legado.
- **SC-002**: El golden master pasa ejecutado contra `app/catalog_engine/` sin necesidad de
  regenerar `pricing_consumption.master.json`.
- **SC-003**: La batería de equivalencia comparativa (semilla fija, entre 100 y 200 casos) reporta
  cero diferencias campo a campo entre la implementación legada y `app/catalog_engine/` en el 100%
  de los casos generados.
- **SC-004**: Los siete ficheros consumidores tienen cero líneas modificadas (diff vacío) al
  concluir la conmutación, comparados contra su estado inmediatamente anterior a esta spec.
- **SC-005**: La suite completa de tests del backend (no solo los de catálogo) pasa exactamente
  igual después de la conmutación que antes de empezar la extracción — mismo conjunto de tests en
  verde.
- **SC-006**: El núcleo puro del motor (los dos módulos/ficheros que contienen
  `compute_line_price` y `_exige_maximo`) tiene cero imports de `sqlalchemy`, verificable por
  inspección estática.

## Assumptions

- El conteo de "45 tests" con el que se abrió el encargo no coincide con el conteo verificado por
  función (`def test_...`) en los dos ficheros citados: 25 en `test_catalog_line_pricing.py` + 16
  en `test_catalog_consumption_plan.py` = 41. Esta spec ancla sus criterios de aceptación al
  conjunto completo de tests de ambos ficheros (sea cual sea su número exacto en el momento de
  ejecutar la extracción), no a un conteo fijo — así que la discrepancia no bloquea ni cambia el
  alcance.
- El encargo describe "los otros ocho" adaptadores; el conteo verificado por firma (funciones que
  reciben `db: Session`) es once: las cuatro de `line_pricing.py`
  (`load_valid_options`, `check_availability`, `validate_option_selection`,
  `grupos_que_descuentan`) más las siete de `consumption_plan.py` (`load_recipe`,
  `load_variant_groups`, `plan_line_consumption`, `required_consumption`, `group_discounts`,
  `variant_label`, `ensure_lines_consume_inventory`). El requisito sustantivo — separar el núcleo
  puro (`compute_line_price`, `_exige_maximo`) del resto, que sí puede depender de SQLAlchemy —
  no depende de este conteo, así que tampoco bloquea el alcance.
- `app/catalog_engine/` es la ruta de destino salvo que la convención real del proyecto indique
  otra; de ser así, se ajusta al mismo nivel que los demás paquetes bajo `app/` (por ejemplo,
  junto a `app/models/`, `app/core/`), conservando el nombre del paquete como referencia estable
  para los imports internos que se creen durante la extracción.
- Ningún módulo distinto al motor de catálogo (`promotions`, `inventory`, `orders`, `sales`,
  `cart`, etc.) se toca en esta spec, incluso si comparte alguna anomalía relacionada (por
  ejemplo, A-04 vive en `orders/consolidation.py`, fuera de alcance).
- La batería comparativa de la Historia 2 corre en un entorno de test aislado (misma base de
  datos de fixture para ambas corridas, sin efectos secundarios sobre datos de producción), igual
  que corren hoy los characterization tests y el golden master.
- El criterio de "igualdad exacta campo a campo" en la Historia 2 incluye tanto los valores de
  retorno de las funciones puras (por ejemplo, el `Decimal` de `compute_line_price`) como el tipo
  y contenido de cualquier excepción levantada (por ejemplo, un `HTTPException` con el mismo
  `status_code` y `detail`) — una diferencia en el mensaje de error o en el código de estado
  cuenta como diferencia, no solo una diferencia en el valor calculado.
