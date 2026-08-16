# Feature Specification: Validación de grupos de opciones (selección y tolerancia de migración)

**Feature Branch**: `004-validacion-grupos-opciones`

**Created**: 2026-08-16

**Status**: Draft

**Naturaleza de esta spec**: **ingeniería inversa / characterization spec**. No describe una
funcionalidad nueva: documenta el comportamiento que el sistema **ya tiene hoy** en
`pos-backend/app/api/v1/catalog/line_pricing.py` (`load_valid_options`,
`validate_option_selection`, `grupos_que_descuentan`) y `app/core/config.py`
(`STRICT_OPTION_SELECTION`), para que sirva de contrato formal de cara a la modernización
(Principio I y Principio III de la [Constitución](../../.specify/memory/constitution.md)). Es el
dominio de la validación de **forma** de la selección (cuántas opciones, de qué grupos, con qué
tolerancia) — distinto del **consumo** que esa selección genera, ya especificado en la spec 003
(`003-consumo-de-inventario-por-receta-y-opcion`). Incluye una regla protegida (A-05) que documenta
una tolerancia de migración deliberada, y dos anomalías (A-06, A-32) que se documentan **sin
especificar como contrato** porque su clasificación sigue `PENDIENTE`, con su tratamiento acordado
citado de `registro-de-anomalias.md`.

**Input**: User description: "Spec de ingeniería inversa: documenta el comportamiento EXISTENTE de
la validación de grupos de opciones (sabores/toppings/extras) del sistema POS Heladería, incluida
la tolerancia deliberada de migración del catálogo histórico, tomado de `reglas-de-negocio.md`
(RN-CAT-27 a RN-CAT-33, RN-CAT-36 a RN-CAT-39) y de `registro-de-anomalias.md` (A-05, A-06, A-32),
para que sirva de contrato en la modernización. No es una feature nueva."

## User Scenarios & Testing *(mandatory)*

<!--
  Cada escenario documenta un comportamiento OBSERVADO en `catalog/line_pricing.py` y en el
  fragmento de `catalog/router.py` que administra los grupos de opciones, no uno deseado. Las
  anomalías conocidas se marcan inline con su tratamiento acordado
  (registro-de-anomalias.md). A-05 es la regla más importante de esta spec: es una tolerancia de
  migración deliberada y protegida, sin characterization test dedicado hoy — el gap se documenta
  explícitamente en Success Criteria.
-->

### User Story 1 - Grupo obligatorio que descuenta inventario exige EXACTAMENTE el máximo, siempre (Priority: P1)

Un cajero vende una presentación con un grupo de opciones obligatorio que además descuenta
inventario (por ejemplo, «Sabores» de una copa de tres bolas). El sistema exige que el cliente
elija exactamente el número máximo permitido — ni menos, aunque cumpla el mínimo, ni más — y
rechaza cualquier desviación, sin importar el estado del flag global de tolerancia.

**Why this priority**: es la guarda que evita el defecto más costoso del módulo: descontar de
menos (sirviendo tres bolas mientras solo descuenta una) o de más. El propio código lo explica con
un ejemplo directo: "un helado de tres bolas... sirve tres y descuenta una" si solo exigiera el
mínimo. Por eso esta regla es bloqueante siempre, incluso con la tolerancia global activada (ver
User Story 3).

**Independent Test**: se puede probar invocando `validate_option_selection` (o el endpoint que la
envuelve) con una variante cuyo grupo obligatorio define `min_select=3, max_select=3,
quantity_per_option>0`, variando el número de opciones elegidas.

**Acceptance Scenarios**:

1. **Given** «Sabores» de «Copa Grande» (`min_select=3, max_select=3, quantity_per_option=120`),
   **When** el cliente elige solo 1 sabor, **Then** el sistema rechaza con un mensaje que "exige
   exactamente 3 opción(es), se enviaron 1" — bloqueante siempre, sin importar
   `STRICT_OPTION_SELECTION` (`RN-CAT-28`, `RN-CAT-30`).
2. **Given** el mismo grupo, **When** el cliente elige 4 sabores (más del máximo), **Then** el
   sistema rechaza con el mismo criterio de "exactamente el máximo" — de más también se rechaza,
   no solo de menos (`RN-CAT-28`).
3. **Given** el mismo grupo, **When** el cliente no elige ningún sabor, **Then** el sistema exige
   el número exacto correspondiente al grupo — `max_select` porque el grupo descuenta inventario,
   no `min_select` (`RN-CAT-29`).
4. **Given** el mismo grupo obligatorio que descuenta inventario, **When** la selección viola
   `min_select`/`max_select` **con `STRICT_OPTION_SELECTION=False`** (el valor por defecto),
   **Then** el rechazo ocurre exactamente igual — el flag global de tolerancia **no** tiene efecto
   sobre grupos que descuentan inventario (`RN-CAT-30`).

---

### User Story 2 - Grupo normal: cualquier cantidad entre el mínimo y el máximo, ambos inclusive (Priority: P1)

Un cajero vende una presentación con un grupo de opciones que no es "obligatorio que además
descuenta inventario" (por ejemplo, un grupo de toppings opcional, o un grupo obligatorio que no
mueve inventario). El sistema permite cualquier cantidad de opciones elegidas dentro del rango
configurado, ambos extremos incluidos.

**Why this priority**: es el comportamiento base contra el que se entiende la excepción de User
Story 1 — sin esta distinción, "obligatorio que exige el máximo" parecería la regla general en vez
de un caso especial reservado a grupos que descuentan inventario.

**Independent Test**: se puede probar invocando `validate_option_selection` con un grupo
`min_select=1, max_select=2` que **no** define `quantity_per_option` (no descuenta inventario),
variando el número de opciones elegidas entre 0 y 3.

**Acceptance Scenarios**:

1. **Given** un grupo «Toppings» (`min_select=1, max_select=2`, sin consumo de inventario),
   **When** el cliente elige 1 opción, **Then** la selección es válida — cumple el mínimo
   (`RN-CAT-27`).
2. **Given** el mismo grupo, **When** el cliente elige 2 opciones, **Then** la selección es
   válida — cumple el máximo, ambos extremos del rango son aceptables (`RN-CAT-27`).
3. **Given** el mismo grupo, **When** el cliente elige 3 opciones, **Then** el sistema rechaza con
   un mensaje que "admite como máximo 2 opción(es), se enviaron 3" (`RN-CAT-27`).

---

### User Story 3 - [Regla protegida A-05] La tolerancia por defecto nunca valida el catálogo histórico (Priority: P1)

Un administrador de catálogo no ha depurado nunca las combinaciones de opciones cargadas
originalmente en el sistema. El sistema, con su configuración de fábrica
(`STRICT_OPTION_SELECTION=False`), no rechaza selecciones que violen `min_select`/`max_select` en
grupos que **no** descuentan inventario — solo registra un warning en logs y deja pasar la venta.

**Why this priority**: es la decisión de negocio central de esta spec, con evidencia directa de
intención (`memoria-historica.md` entrada #9, 2026-08-03, commit `03469cad`, Deimer Hernandez):
"el catálogo histórico nunca se validó" — activar el chequeo a ciegas rechazaría combinaciones ya
cargadas en producción. **Confirmado en entrevista de negocio (P18)**: el administrador de
catálogo respondió que el catálogo **nunca se depuró** — el script `opciones_fuera_de_grupo.py`,
dejado explícitamente para ese propósito, nunca se corrió. El default `False` debe seguir así.

**Independent Test**: se puede probar invocando `validate_option_selection` con
`STRICT_OPTION_SELECTION=False` (el valor de fábrica, sin override en `.env.example`) sobre un
grupo que no descuenta inventario con una selección fuera de rango, y verificando que la operación
continúa con un warning en logs en vez de un error.

**Acceptance Scenarios**:

1. **Given** un grupo «Toppings» (`min_select=1, max_select=2`, sin consumo de inventario) y
   `STRICT_OPTION_SELECTION=False`, **When** el cliente no elige ninguna opción (viola el
   mínimo), **Then** la operación **continúa** — se registra un warning en logs, no se lanza
   error (`RN-CAT-31`).
2. **Given** el valor real de `STRICT_OPTION_SELECTION` en el archivo `.env.example` del
   repositorio, **When** se inspecciona ese archivo, **Then** **no existe ninguna línea que
   sobrescriba** el valor por defecto — el default de fábrica (`False`) es el que efectivamente
   corre en cualquier entorno que parta de esa plantilla (`app/core/config.py:62`).
3. **Given** la pregunta de negocio pendiente citada en `registro-de-anomalias.md` (A-05) —
   "¿se llegó a correr `opciones_fuera_de_grupo.py` alguna vez?" —, **When** se consulta la
   entrevista de negocio (P18), **Then** la respuesta registrada es "no, nunca se ha hecho",
   lo que **cierra** esa decisión pendiente a favor de mantener el default `False`.

**Regla protegida A-05 — `INTENCIONAL [PROTEGIDA]`**: esta spec la especifica **tal cual, sin
tocar el default**, siguiendo el tratamiento acordado en `registro-de-anomalias.md`. **Gap de
caracterización**: no existe hoy ningún characterization test dedicado a
`load_valid_options`/`STRICT_OPTION_SELECTION` de forma aislada — es una brecha prioritaria a
cerrar antes de poder proteger esta regla con un test explícito, dado que es una regla `PROTEGIDA`
sin caracterización (ver Success Criteria, SC-002).

---

### User Story 4 - Las violaciones en grupos que descuentan inventario nunca se toleran (Priority: P1)

Un cajero vende una presentación con un grupo de opciones que descuenta inventario, sin importar
si ese grupo es obligatorio o no. Cualquier violación de `min_select`/`max_select` en ese grupo
detiene la venta con un rechazo, incluso con la tolerancia global desactivada.

**Why this priority**: es la contraparte directa de User Story 3 — explica por qué la tolerancia
de migración (A-05) es segura: nunca se extiende a los grupos que mueven inventario. El propio
código lo declara con un ejemplo: "aceptar cinco opciones en un grupo `max_select=1` descuenta
cinco veces el helado".

**Independent Test**: se puede probar con el mismo grupo de User Story 1 (obligatorio, descuenta
inventario) y `STRICT_OPTION_SELECTION=False`, verificando que la violación bloquea igual que con
el flag en `True`.

**Acceptance Scenarios**:

1. **Given** un grupo que descuenta inventario (con o sin ser obligatorio) y
   `STRICT_OPTION_SELECTION=False`, **When** la selección viola `min_select` o `max_select`,
   **Then** el sistema rechaza la venta — el flag de tolerancia global **solo** aplica a grupos
   que no descuentan inventario, nunca a estos (`RN-CAT-30`, `RN-CAT-31`).

---

### User Story 5 - [Anomalía A-06, `PENDIENTE`, riesgo aceptado] Una opción de un grupo ajeno a la variante se cuela con la tolerancia activa (Priority: P2)

Un cajero envía, junto con la selección normal, una opción que pertenece a un grupo que la
variante vendida **no** ofrece (por ejemplo, un «Extra de Pizza» sobre un «Cono Simple» que solo
ofrece «Sabores»). Con `STRICT_OPTION_SELECTION=False`, el sistema no rechaza esa opción: sigue
sumando su `extra_price` al cobro y, si tiene insumo propio, sigue generando consumo de ese
insumo ajeno a la variante vendida.

**Why this priority**: es la consecuencia directa y ya identificada de la tolerancia de User
Story 3 — el mismo mecanismo que protege el catálogo histórico también deja pasar este caso, que
nadie confirmó si era un residuo previsto o un vector no contemplado al diseñar la tolerancia.

**Independent Test**: se puede probar enviando, junto con la selección válida, una opción cuyo
`option_group_id` no corresponde a ningún grupo vinculado a la variante, con
`STRICT_OPTION_SELECTION=False`.

**Acceptance Scenarios**:

1. **Given** «Cono Simple» que solo ofrece el grupo «Sabores», **When** se envía además una
   opción «Extra de Pepperoni» de un grupo «Extras de Pizza» (`extra_price=3000`,
   `item_quantity=1` ligado a un insumo de pizza) con `STRICT_OPTION_SELECTION=False`, **Then**
   la venta **no se rechaza**: cobra los $3.000 extra y genera consumo del insumo de pizza, pese
   a que «Cono Simple» nunca ofreció ese grupo (`RN-CAT-32`).
2. **Given** el mismo escenario, **When** se compara contra `STRICT_OPTION_SELECTION=True`,
   **Then** este caso **tampoco** queda cubierto por el flag estricto por sí solo — pertenecer a
   un grupo ajeno nunca es, por sí mismo, un problema bloqueante en la validación de selección
   (`RN-CAT-32`).

**Anomalía A-06 — clasificación `PENDIENTE`, tratamiento cerrado (ronda 2, pregunta P7-bis)**:
esta situación sigue sin reunir el testimonio de negocio necesario para clasificarse como
`INTENCIONAL` o `ACCIDENTAL` en sentido estricto. Pero en la segunda ronda de entrevista
(P7-bis, dirigida a dueño/gerente) el tratamiento quedó decidido explícitamente: **"aceptar el
riesgo por ahora, mientras el catálogo no esté depurado (igual que A-05), dejarlo así, no es
prioridad"** — no se prioriza corrección ni consulta a datos, con el mismo criterio ya aceptado
para A-05. Esta spec documenta el comportamiento observado tal cual, sin fijarlo como corrección
obligatoria de la modernización.

---

### User Story 6 - [Mecanismo raíz de A-04] La validación de selección es opcional para el llamador (Priority: P1)

Un desarrollador integra un nuevo punto de entrada que arma una línea de venta con opciones
elegidas. La función que carga y valida esas opciones (`load_valid_options`) solo aplica las
reglas de `min_select`/`max_select`/pertenencia si se le pasa explícitamente la variante que se
está vendiendo; si se omite ese parámetro, las opciones se cargan sin ninguna validación de forma.

**Why this priority**: esta regla, por sí sola, no es un bug — es una decisión de diseño de la
función. Pero es el mecanismo **exacto** que permite el bug real documentado en la anomalía A-04:
el camino que usa realmente el mesero en la terminal (`add_item_to_table` → `consolidation.py`)
no pasa `variant`, así que hoy vende sin validar ninguna de las reglas de User Story 1 a 4. Esta
spec fija la regla en sí; la spec 009 (fuera de alcance aquí) documenta y corrige ese caller real.

**Independent Test**: se puede probar invocando `load_valid_options` dos veces con el mismo
conjunto de opciones fuera de rango — una vez pasando `variant`, otra vez sin pasarlo — y
comparando si se aplica o no la validación.

**Acceptance Scenarios**:

1. **Given** una lista de opciones elegidas que violaría `min_select`/`max_select` de un grupo
   obligatorio que descuenta inventario, **When** se invoca `load_valid_options` **pasando**
   `variant`, **Then** se aplican todas las reglas de User Story 1 a 4 sobre esa selección
   (`RN-CAT-33`).
2. **Given** la misma lista de opciones, **When** se invoca `load_valid_options` **sin pasar**
   `variant`, **Then** las opciones se cargan sin ninguna validación de `min_select`, `max_select`
   ni pertenencia al grupo — ni siquiera las reglas bloqueantes de User Story 1 y 4 se aplican
   (`RN-CAT-33`).
3. **Given** el docstring de `load_valid_options`, **When** se lee su advertencia, **Then**
   reconoce el riesgo sin resolverlo: "pasarla siempre que se pueda" es una advertencia, no una
   garantía exigida por el código (`RN-CAT-33`).

**Nota de alcance — anomalía A-04**: el caller real que efectivamente omite `variant`
(`app/api/v1/orders/consolidation.py:199`, camino del mesero) y su corrección (pasar
`variant=variant`, ya hecha una vez en el commit `03469ca` y perdida en el merge posterior
`ee94f30`) están **fuera de alcance de esta spec** — se documentan y se corrigen en la spec 009.
Esta spec únicamente fija que la validación es opcional por diseño, no obligatoria por contrato.

---

### User Story 7 - Administración del catálogo de grupos de opciones (Priority: P3)

Un administrador de catálogo gestiona los grupos de opciones y sus opciones desde el panel: intenta
desactivar o eliminar un grupo en uso, crea opciones con nombres repetidos, o desvincula el insumo
de una opción.

**Why this priority**: son tres guardas independientes de integridad del catálogo, de menor
frecuencia operativa que la validación de venta (User Story 1 a 6), pero necesarias para que el
catálogo no quede en un estado inconsistente.

**Independent Test**: cada una de las tres reglas se puede probar de forma aislada contra los
endpoints de administración de grupos y opciones (`app/api/v1/catalog/router.py`), sin depender de
una venta real.

**Acceptance Scenarios**:

1. **Given** un grupo «Sabores» asignado a «Copa Grande» y «Copa Mediana», **When** un
   administrador intenta desactivarlo o eliminarlo, **Then** el sistema rechaza con un mensaje que
   lista las presentaciones que aún lo ofrecen (p. ej. "«Helado · Copa Grande», «Helado · Copa
   Mediana» lo ofrece a sus clientes. Quítalo de esas presentaciones primero") (`RN-CAT-36`).
2. **Given** un grupo «Sabores» con una opción «Fresa», **When** un administrador intenta crear
   otra opción llamada «Fresa» **dentro del mismo grupo**, **Then** el sistema rechaza por nombre
   duplicado (`RN-CAT-37`).
3. **Given** un grupo «Sabores» con una opción «Fresa» y un grupo distinto «Toppings» **sin**
   ninguna opción llamada «Fresa», **When** un administrador crea una opción «Fresa» en
   «Toppings», **Then** el sistema **permite** la creación — el nombre único aplica dentro del
   grupo, no globalmente (`RN-CAT-37`).
4. **Given** una opción con `inventory_item_id` e `item_quantity=80` configurados, **When** un
   administrador envía `PATCH {"inventory_item_id": null, "item_quantity": 999}` en el mismo
   request, **Then** el resultado final es `item_quantity=0`, no `999` — desvincular el insumo
   resetea forzosamente la cantidad, sin importar qué valor traiga el resto del mismo payload
   (`RN-CAT-38`).

---

### User Story 8 - [Anomalía A-32, `PENDIENTE`, documentada sin especificar] Dos criterios distintos para "¿este grupo descuenta inventario?" (Priority: P3)

Un administrador de catálogo configura una opción con cantidad de consumo puesta
(`item_quantity>0`) pero sin insumo de inventario enlazado (catálogo a medio terminar, o insumo
desactivado). Según qué función del sistema evalúe "¿este grupo descuenta inventario?", la
respuesta difiere: la que valida la selección al armar la venta dice que sí; la que confirma la
venta dice que no.

**Why this priority**: es una discrepancia interna entre dos piezas de código que resuelven, en
la práctica, la misma pregunta de negocio, sin que ningún comentario reconozca ni justifique la
diferencia — pero coinciden siempre que el catálogo esté bien formado, así que su impacto
práctico depende de datos que hoy no están disponibles.

**Independent Test**: se puede probar comparando el resultado de `grupos_que_descuentan`
(`app/api/v1/catalog/line_pricing.py`) y `group_discounts` (`app/api/v1/catalog/consumption_plan.py`)
sobre el mismo grupo con una opción `item_quantity=10, inventory_item_id=None`.

**Acceptance Scenarios (comportamiento observado, no especificado como contrato)**:

1. **Given** una opción «Extra Dulce» con `item_quantity=10` e `inventory_item_id=None`, **When**
   `grupos_que_descuentan` (usada en la validación de selección al alta) evalúa su grupo,
   **Then** lo clasifica como "grupo que descuenta inventario" — exige solo `item_quantity>0`
   (`RN-CAT-39`).
2. **Given** la misma opción y el mismo grupo, **When** `group_discounts` (usada en
   `ensure_lines_consume_inventory` al confirmar la venta) evalúa ese grupo, **Then** lo
   clasifica como "grupo que NO descuenta inventario" — exige además `active=True` e
   `inventory_item_id` no nulo (`RN-CAT-39`).
3. **Given** esa discrepancia, **When** un administrador intenta vender una variante configurada
   así, **Then** puede recibir el mensaje de error equivocado ("no tiene receta configurada")
   cuando el problema real es un insumo no enlazado, no la ausencia de configuración.

**Anomalía A-32 — clasificación `PENDIENTE`, requiere arqueología de datos**: esta spec documenta
las dos definiciones tal cual existen en el código, sin unificarlas ni fijar cuál es la correcta.
**Pregunta que la desbloquearía** (candidata explícita a consulta de datos, no bloquea esta spec):
¿existen hoy en el catálogo real opciones con `item_quantity` puesto pero sin insumo enlazado (o
inactivas) que sigan asignadas a algún grupo de alguna variante vendible? **Tratamiento acordado**:
documentar sin especificar hasta tener el dato; si se confirma que existen casos así, unificar el
criterio en fase de modernización.

---

### Edge Cases

- **Grupo obligatorio (`min_select>0`) que NO descuenta inventario**: le aplica el rango
  `[min_select, max_select]` inclusive de User Story 2, no la exigencia de "exactamente el
  máximo" de User Story 1 — esa exigencia depende de que el grupo descuente inventario, no solo
  de que sea obligatorio (`RN-CAT-27` vs `RN-CAT-28`).
- **`STRICT_OPTION_SELECTION=True` en un entorno que sí depuró su catálogo**: queda fuera de la
  evidencia disponible hoy — no hay `.env.example` ni entorno conocido que lo active; esta spec
  documenta el comportamiento del default `False`, no un escenario hipotético con el flag
  encendido salvo donde ya se contrasta explícitamente (User Story 1, escenario 4).
- **Opción de un grupo ajeno que además viola su propio `min_select`/`max_select`**: no cubierto
  por ninguna regla `RN-CAT` de este recorte — la pertenencia al grupo (`RN-CAT-32`) y el rango de
  selección (`RN-CAT-27`/`RN-CAT-28`) son chequeos independientes en el código citado.
- **Renombrar una opción a un nombre que ya existe en su propio grupo**: mismo rechazo por nombre
  duplicado que crear una opción nueva — el scope compuesto `(option_group_id, name)` aplica a
  cualquier operación que fije el nombre, no solo a la creación (`RN-CAT-37`).
- **`PATCH` que envía `inventory_item_id` con un valor no nulo distinto al actual, junto con
  `item_quantity`**: fuera de alcance de `RN-CAT-38`, que solo documenta el caso específico de
  `inventory_item_id: null` — el reseteo forzado a 0 solo se activa por esa desvinculación
  explícita.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: Para un grupo de opciones que no exige el máximo (ver FR-002), la cantidad de
  opciones elegidas DEBE estar en el rango `[min_select, max_select]`, ambos extremos inclusive
  (`RN-CAT-27`).
- **FR-002**: Un grupo de opciones obligatorio (`min_select>0`) que además descuenta inventario
  DEBE exigir que se elija exactamente `max_select` opciones — ni menos (aunque cumpla el
  mínimo) ni más se acepta (`RN-CAT-28`).
- **FR-003**: Si no se elige ninguna opción de un grupo obligatorio, el sistema DEBE pedir el
  número exacto correspondiente: `max_select` si el grupo descuenta inventario, `min_select` si
  no (`RN-CAT-29`).
- **FR-004**: Cualquier violación de selección en un grupo que descuenta inventario DEBE ser
  bloqueante, sin excepción y sin importar el valor de `STRICT_OPTION_SELECTION` (`RN-CAT-30`).
- **FR-005 [Regla protegida A-05]**: `STRICT_OPTION_SELECTION` DEBE mantener su valor por defecto
  `False` (sin override en `.env.example`), y con ese valor el sistema DEBE tolerar violaciones de
  `min_select`/`max_select` **únicamente** en grupos que no descuentan inventario, registrando un
  warning en logs en vez de rechazar la operación. Esta tolerancia es deliberada para no romper el
  catálogo histórico nunca depurado (`memoria-historica.md` #9; confirmado en entrevista P18: el
  script `opciones_fuera_de_grupo.py` nunca se corrió) y DEBE especificarse tal cual, sin cambiar
  el default, mientras el negocio no confirme que el catálogo ya fue depurado (`RN-CAT-31`).
- **FR-006 [Anomalía A-06, `PENDIENTE` — documentada sin especificar como contrato obligatorio,
  tratamiento de riesgo aceptado en P7-bis]**: con `STRICT_OPTION_SELECTION=False`, elegir una
  opción de un grupo que la variante vendida no ofrece actualmente **no se rechaza**: sigue
  sumando su `extra_price` al cobro y, si tiene insumo propio, sigue generando consumo de ese
  insumo. Esta spec documenta el comportamiento observado; el tratamiento acordado es aceptar el
  riesgo por ahora, sin priorizar corrección ni consulta a datos, mismo criterio que FR-005
  (`RN-CAT-32`).
- **FR-007 [Mecanismo raíz de A-04, corregido en spec 009]**: la validación de
  `min_select`/`max_select`/pertenencia al grupo (`load_valid_options`) DEBE aplicarse únicamente
  cuando el llamador pasa el parámetro `variant`; si se omite, las opciones se cargan sin ninguna
  validación de forma. Esta spec fija la regla tal cual existe hoy; no corrige ningún caller
  específico que la omita (`RN-CAT-33`).
- **FR-008**: Un grupo de opciones NO DEBE poder desactivarse ni eliminarse mientras exista al
  menos una variante que todavía lo tenga asignado (`RN-CAT-36`).
- **FR-009**: El nombre de una opción DEBE ser único dentro de su propio grupo; el mismo nombre
  SÍ puede repetirse en grupos distintos (`RN-CAT-37`).
- **FR-010**: Al desvincular el insumo de una opción (`inventory_item_id: null` explícito), el
  sistema DEBE resetear `item_quantity` a `0`, sin importar qué otro valor de `item_quantity`
  traiga el mismo request (`RN-CAT-38`).
- **FR-011 [Anomalía A-32, `PENDIENTE` — documentada sin especificar como contrato]**: existen hoy
  dos funciones distintas que responden "¿este grupo descuenta inventario?" con criterios
  diferentes — la usada en la validación de selección al alta exige solo `item_quantity>0`; la
  usada al confirmar la venta exige además `active=True` e `inventory_item_id` no nulo. Esta spec
  documenta ambos criterios tal cual existen, sin unificarlos, hasta que una consulta a datos
  confirme si la discrepancia tiene impacto real en el catálogo vigente (`RN-CAT-39`).

### Key Entities *(include if feature involves data)*

- **OptionGroup**: grupo de opciones (p. ej. «Sabores», «Toppings»). Atributos relevantes a esta
  spec: `min_select`, `max_select`, `active`. Puede estar vinculado a más de una variante
  (`RN-CAT-36`).
- **VariantOptionGroup**: vínculo entre una variante y un grupo de opciones. Determina si, para
  esa variante, el grupo es obligatorio y si descuenta inventario (a través de
  `quantity_per_option`, fuera de alcance de esta spec — ver spec 003).
- **Option**: valor dentro de un grupo (p. ej. un sabor). Atributos relevantes a esta spec:
  `name` (único dentro de su `option_group_id`, `RN-CAT-37`), `option_group_id`, `extra_price`,
  `inventory_item_id` (opcional), `item_quantity` (forzado a `0` si se desvincula el insumo,
  `RN-CAT-38`), `active`.
- **`STRICT_OPTION_SELECTION`**: bandera de configuración global (`app/core/config.py`), valor por
  defecto `False`, sin override en `.env.example`. Determina si las violaciones de
  `min_select`/`max_select` en grupos que **no** descuentan inventario se toleran (`False`, warning
  en logs) o se rechazan (`True`). Nunca afecta a grupos que descuentan inventario (`RN-CAT-30`,
  `RN-CAT-31`).

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: El 100% de las reglas `RN-CAT-27` a `RN-CAT-33` y `RN-CAT-36` a `RN-CAT-39` puede
  verificarse ejecutando los pasos descritos en esta spec contra un `pos-backend` en ejecución,
  sin necesitar leer `line_pricing.py` ni `config.py` para entender el comportamiento esperado.
- **SC-002 [Gap de caracterización, prioritario]**: no existe hoy ningún characterization test
  dedicado a `load_valid_options`/`validate_option_selection`/`STRICT_OPTION_SELECTION` de forma
  aislada (`test_variant_option_groups.py`, el script más cercano, cubre consumo — spec 003, no
  esta forma de selección). Cerrar este gap es prioritario antes de poder proteger la regla A-05
  (`FR-005`) con un test explícito, dado que es una regla `PROTEGIDA` sin caracterización.
- **SC-003**: La decisión de negocio pendiente citada en `registro-de-anomalias.md` para A-05
  (¿se corrió alguna vez `opciones_fuera_de_grupo.py`?) queda **cerrada** por esta spec, citando
  la entrevista P18 ("no, nunca se ha hecho") como evidencia de que el default `False` debe
  seguir sin cambios hasta nueva decisión explícita del negocio.
- **SC-004**: Las dos anomalías `PENDIENTE` de esta spec (A-06, A-32) quedan documentadas con su
  comportamiento observado, su evidencia de código y su tratamiento acordado (riesgo aceptado
  para A-06 vía P7-bis; documentar sin especificar para A-32 hasta arqueología de datos), de
  forma que el equipo de modernización no las reintroduzca por accidente ni las trate como si ya
  tuvieran una decisión de negocio distinta a la registrada.
- **SC-005**: Ningún administrador puede desactivar o eliminar un grupo de opciones todavía
  ofrecido por una variante, ni crear dos opciones con el mismo nombre dentro del mismo grupo — el
  100% de esos intentos recibe un rechazo con un mensaje que identifica la causa.

## Out of Scope

- **Qué se descuenta y cuánto una vez que la selección es válida** (cálculo de consumo por receta
  y por opción, `plan_line_consumption`) — cubierto por la spec 003
  (`003-consumo-de-inventario-por-receta-y-opcion`).
- **El caller real que omite pasar `variant` a `load_valid_options`** (`add_item_to_table` →
  `consolidation.py:199`, camino real del mesero, anomalía A-04) y su corrección — esta spec fija
  la regla de que la validación es opcional por diseño (`RN-CAT-33`, FR-007); el caller concreto
  que la omite y su corrección se documentan en la spec 009, aún no escrita en este
  reconocimiento.
- **El precio y el SKU de la variante** — cubierto por la spec 002
  (`002-catalogo-productos-variantes-y-precios`).

## Assumptions

- **Esta es una spec de ingeniería inversa, no de una feature nueva**: a diferencia del resto de
  las guías de este template ("evitar detalles de implementación"), aquí los nombres de campo,
  valores por defecto, mensajes de error y funciones citadas **son** el contrato observable que se
  está documentando — se citan explícitamente porque los criterios de aceptación deben ser
  verificables directamente contra `pos-backend` en ejecución.
- **A-05 se especifica como regla protegida, tal cual, sin cambiar el default**: a diferencia de
  la mayoría de reglas `INTENCIONAL` de este proyecto, esta tiene evidencia directa de decisión de
  negocio (memoria histórica + entrevista P18) y su reversión sin depurar antes el catálogo
  rechazaría ventas legítimas de la noche a la mañana. Cualquier cambio de este default requiere
  una nueva decisión de negocio explícita, no una decisión técnica unilateral.
- **A-06 y A-32 se documentan pero NO se especifican como contrato obligatorio**: siguiendo
  instrucción explícita de alcance, estas dos anomalías quedan con clasificación `PENDIENTE` en
  `registro-de-anomalias.md` — se describe el comportamiento observado hoy (porque esta spec
  documenta "lo que el sistema YA hace"), pero no se fija como el comportamiento correcto ni
  obligatorio para la modernización. A-06 tiene tratamiento de riesgo aceptado ya decidido
  (P7-bis); A-32 sigue pendiente de arqueología de datos.
- **El gap de caracterización (SC-002) es una brecha documentada, no una tarea de esta spec**: esta
  spec no crea el characterization test faltante — señala su ausencia como prioridad para poder
  proteger A-05 con evidencia de test explícita en el futuro.
- **Los valores numéricos y de ejemplo citados en los escenarios** (`min_select=3, max_select=3`,
  `extra_price=3000`, nombres «Copa Grande», «Cono Simple», «Extra de Pepperoni») son
  ilustrativos, tomados de `reglas-de-negocio.md` y de `registro-de-anomalias.md` — no representan
  necesariamente el catálogo real vigente hoy en producción.
