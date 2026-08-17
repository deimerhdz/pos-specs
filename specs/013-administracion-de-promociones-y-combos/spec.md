# Feature Specification: Administración de promociones y combos — creación, edición, validación de forma y máquina de estados

**Feature Branch**: `013-administracion-de-promociones-y-combos`

**Created**: 2026-08-16

**Status**: Draft

**Naturaleza de esta spec**: **ingeniería inversa / characterization spec**. No describe una
funcionalidad nueva: documenta el comportamiento que el sistema **ya tiene hoy** en
`pos-backend/app/api/v1/promotions/router.py`, `schemas.py` y `app/core/scheduler.py`
(RN-PROMO-46 a RN-PROMO-78, RN-SCHED-10/11), para que sirva de contrato formal de cara a la
modernización (Principio I y Principio III de la
[Constitución](../../.specify/memory/constitution.md)). Es el dominio de "cómo se crea, edita
y valida la forma de una promoción o combo, y qué transiciones de estado tiene permitidas" —
complementario a la spec 012 (que documenta cómo se **calcula** el descuento una vez la
promoción ya existe y está `active`). Incluye un hallazgo `ACCIDENTAL`+`PENDIENTE` con impacto
real (A-30: dos vectores de `IntegrityError` no controlado en `PATCH`), una porción `PENDIENTE`
sin decisión de negocio (A-37, la parte de máquina de estados del cluster ya introducido en la
spec 012), y una discrepancia `ACCIDENTAL` de bajo impacto entre el job de medianoche y el
criterio de vigencia real (A-39, ya introducida en la spec 012 desde el lado del motor de
cálculo; aquí se documenta desde el lado del job en sí).

**Input**: User description: "Spec de ingeniería inversa: documenta el comportamiento EXISTENTE
de la administración de promociones y combos (creación, edición, validación de forma, máquina de
estados) del sistema POS Heladería — RN-PROMO-46 a RN-PROMO-78, RN-SCHED-10/11, tomado de
`pos-backend/app/api/v1/promotions/router.py`, `schemas.py`, `app/core/scheduler.py`, para que
sirva de contrato en la modernización."

## User Scenarios & Testing *(mandatory)*

<!--
  Cada escenario documenta un comportamiento OBSERVADO en `promotions/router.py`,
  `promotions/schemas.py`, `promotions/service.py` (porción CRUD/máquina de estados) y
  `core/scheduler.py`, no uno deseado. Las anomalías conocidas se marcan inline con su
  tratamiento acordado (`registro-de-anomalias.md`).
-->

### User Story 1 - Solo `active` habilita el descuento automático; la máquina de estados tiene transiciones fijas (Priority: P1)

`PROMOTION_TRANSITIONS` define un mapa fijo por estado origen: `draft→{active, finished}`,
`active→{paused, finished}`, `paused→{active, finished}`, `finished→{}` (terminal). Cualquier
transición fuera de ese mapa se rechaza con 409. Solo el estado `active` habilita que
`_valid_now` considere la promoción como candidata de descuento (spec 012, primer paso del AND
con cortocircuito).

**Why this priority**: es la puerta de entrada a todo lo demás — una promoción mal
transicionada nunca debe llegar a cobrar, y una transición no controlada podría dejar el
catálogo de promociones en un estado inconsistente (por ejemplo, revivir una `finished`).

**Independent Test**: se puede probar invocando `change_status(db, promo, new_status)` con cada
par (origen, destino) posible entre los cuatro estados, sin pasar por un cobro real.

**Acceptance Scenarios**:

1. **Given** una promoción `status="finished"`, **When** se pide `PATCH /status
   {"status":"active"}`, **Then** el sistema rechaza con 409 "Transición no permitida:
   finished -> active" — `finished` es terminal (`RN-PROMO-53`).
2. **Given** una promoción `status="draft"`, **When** se pide `PATCH /status
   {"status":"paused"}`, **Then** el sistema rechaza con 409 — `paused` no está entre los
   destinos permitidos desde `draft` (`RN-PROMO-53`).
3. **Given** una promoción `status="paused"` vigente por fecha/hora/día, **When** se evalúa si
   habilita el descuento automático, **Then** NO lo habilita — solo `status="active"` pasa el
   primer paso del AND de vigencia (`RN-PROMO-46`, compartido conceptualmente con la spec 012).
4. **Given** una promoción `type="combo"` con solo 1 componente en `combo_items` (por ejemplo,
   porque una variante fue borrada en cascada después de crear el combo), **When** se pide
   `PATCH /status {"status":"active"}`, **Then** el sistema rechaza con 422 — la exigencia de
   `>=2` componentes se revalida en el momento de la transición, no solo en la creación
   (`RN-PROMO-55`).

---

### User Story 2 - El cambio de forma (`type`, `targets`, `combo_items`) solo se permite en `draft` (Priority: P1)

`PATCH /promotions/{id}/shape` es el único endpoint que puede cambiar `type`, `targets` o
`combo_items` de una promoción existente — `PATCH /promotions/{id}` (el PATCH escalar) ni
siquiera acepta esos campos en su schema (`PromotionUpdate`). `update_shape` rechaza con 409
cualquier intento si la promoción no está en `draft`.

**Why this priority**: es la regla que protege la trazabilidad histórica — una promoción que ya
estuvo `active` pudo explicar el descuento de una venta pasada, y cambiarle la forma
reescribiría esa historia sin dejar rastro.

**Independent Test**: se puede probar invocando `update_shape(db, promo, body)` sobre
promociones en cada uno de los cuatro estados, con un payload de cambio de `type` o `targets`,
sin pasar por un cobro real.

**Acceptance Scenarios**:

1. **Given** una promoción `status="active"`, **When** se pide `PATCH /shape {"type":"fixed"}`,
   **Then** el sistema rechaza con 409 "Solo una promoción en borrador puede cambiar de tipo o
   alcance. Duplícala, edita la copia y finaliza la original." (`RN-PROMO-40`, `RN-PROMO-56`).
2. **Given** una promoción `status="draft"`, **When** se pide `PATCH /shape` cambiando `type` y
   `targets` a la vez, **Then** el sistema acepta — las validaciones de forma que dependen del
   tipo (por ejemplo, "el precio por producto solo aplica a `qty_price`") se evalúan contra el
   tipo **ya aplicado**, no el original, porque ambos pueden cambiar en el mismo `PATCH`
   (`service.py:629-630`).
3. **Given** cualquier promoción en cualquier estado, **When** se envía `type`, `targets` o
   `combo_items` en el cuerpo de `PATCH /promotions/{id}` (el PATCH escalar, no `/shape`),
   **Then** esos campos son ignorados silenciosamente — `PromotionUpdate` no los define, así
   que Pydantic los descarta sin error (`RN-PROMO-57`).

---

### User Story 3 - Duplicar siempre crea la copia en `draft`, exige nombre nuevo y único (Priority: P1)

`POST /promotions/{id}/duplicate` es la vía de escape para cambiar la forma de una promoción que
ya no está en `draft`: copia todos los campos (incluidos `targets` y `combo_items`) a una
promoción nueva con `status="draft"`, sin importar el estado de la original. Exige un `name`
distinto y único, validado por el router antes de invocar el servicio.

**Why this priority**: es el flujo operativo real para "corregir" una promoción activa sin
tocar su historia — sin él, User Story 2 dejaría a cualquier promoción ya activada congelada en
su forma original para siempre.

**Independent Test**: se puede probar invocando `duplicate(db, promo, new_name)` sobre
promociones en cada uno de los cuatro estados, verificando que la copia siempre nace `draft`.

**Acceptance Scenarios**:

1. **Given** una promoción `status="finished"` con 2 `targets` y `priority=500`, **When** se
   duplica con un nombre nuevo, **Then** la copia nace `status="draft"`, con los mismos
   `targets`, `priority` y demás campos escalares — el estado de origen nunca se propaga
   (`RN-PROMO-42`, `RN-PROMO-58`).
2. **Given** una promoción `status="active"`, **When** se pide duplicar con un `name` que ya
   existe en otra promoción, **Then** el router rechaza con 409 antes de invocar
   `service.duplicate` — el nombre se valida primero (`RN-PROMO-59`, `RN-PROMO-78`).
3. **Given** un combo con 3 `combo_items`, **When** se duplica, **Then** la copia recibe una
   fila nueva por cada `combo_item` del original, con el mismo `product_variant_id` y
   `quantity` — no comparte filas con el original (`service.py:688-691`).

---

### User Story 4 - Validación de forma en triple capa: porcentaje ≤100, `qty_price` con `min_qty≥2`, `priority` 0-1000, target XOR, combo ≥2 sin `targets` (Priority: P1)

Cinco reglas de forma, cada una reforzada en más de una capa para que un `PATCH` no pueda
saltarse lo que el `POST` de creación sí exige: (1) `value` de una promoción `percent` nunca
puede superar 100 — validado en el schema de creación, en el servicio de `update` (porque
`PromotionUpdate` no lleva `type` y no puede autovalidarse), y en un `CHECK` de PostgreSQL como
última red; (2) `qty_price` exige `min_qty>=2` **a nivel de la promoción**, mismo patrón de
triple capa; (3) `priority` está acotado a `[0, 1000]` en el schema de creación y de
actualización; (4) `TargetIn` exige exactamente uno de `product_id`/`category_id` (XOR estricto
en la aplicación; el `CHECK` de BD es más laxo, solo exige "al menos uno"); (5) un combo exige
al menos 2 `product_variant_id` distintos en `combo_items`, sin duplicados, y no puede combinarse
con `targets`.

**Why this priority**: es el conjunto de reglas que impide que una promoción mal configurada
llegue a cobrar un monto incorrecto — el propio comentario del código narra el incidente real
que motivó la capa de servicio ("un PATCH con `value=500`... tumbaba la caja").

**Independent Test**: cada regla se puede probar con un payload construido a mano contra
`PromotionCreate`/`PromotionUpdate`/`service.update`, sin pasar por un cobro real. La capa de
BD solo se ejercita si las dos anteriores se saltan deliberadamente en un test de integración.

**Acceptance Scenarios**:

1. **Given** `POST /promotions {"type":"percent","value":100.00,...}`, **When** se valida,
   **Then** se acepta — el límite es inclusivo (`RN-PROMO-62`).
2. **Given** `POST /promotions {"type":"percent","value":100.01,...}`, **When** se valida,
   **Then** 422 "Un descuento porcentual no puede superar 100" (`RN-PROMO-62`).
3. **Given** una promoción `percent` ya creada con `value=50`, **When** se pide `PATCH
   {"value":150}`, **Then** 422 en la capa de servicio — el schema de `PromotionUpdate` por sí
   solo no puede rechazarlo porque no conoce el `type` (`RN-PROMO-62`, evidencia:
   `schemas.py:101-105`, `service.py:581-585`, `models/promotion.py:105-108`).
4. **Given** `POST /promotions {"type":"qty_price","min_qty":1,...}`, **When** se valida,
   **Then** 422 "qty_price requiere min_qty >= 2" (`RN-PROMO-63`).
5. **Given** `POST /promotions {"priority":1001,...}` o `{"priority":-1,...}`, **When** se
   valida, **Then** 422 en ambos casos — el rango `[0,1000]` es inclusivo en sus dos extremos
   (`RN-PROMO-73`).
6. **Given** un `TargetIn` con `product_id` y `category_id` ambos presentes, o ambos ausentes,
   **When** se valida, **Then** 422 "Un target apunta a un producto o a una categoría, no a los
   dos" o "Cada target requiere product_id o category_id" respectivamente — exactamente uno de
   los dos es obligatorio (`RN-PROMO-71`).
7. **Given** `POST /promotions {"type":"combo","combo_items":[{"product_variant_id":"X"}],...}`
   (un solo componente), **When** se valida, **Then** 422 "Un combo requiere al menos 2
   productos distintos en combo_items" (`RN-PROMO-72`).
8. **Given** un combo con el mismo `product_variant_id` repetido en `combo_items`, **When** se
   valida, **Then** 422 "combo_items no puede repetir product_variant_id" (`RN-PROMO-72`).
9. **Given** `POST /promotions {"type":"combo","targets":[{"category_id":"X"}],
   "combo_items":[...]}` (combo con `targets` a la vez), **When** se valida, **Then** 422 "Un
   combo define su alcance en combo_items, no en targets" (`RN-PROMO-72`).

---

### User Story 5 - El nombre de la promoción debe ser único en creación, actualización y duplicado (Priority: P2)

`ensure_unique` se invoca en los tres puntos de entrada que pueden fijar un `name`: `POST
/promotions`, `PATCH /promotions/{id}` (solo si `name` viene presente y no es `null`) y `POST
/promotions/{id}/duplicate`. En la actualización, la comprobación excluye la propia promoción
(`exclude_id`) para permitir un `PATCH` que no cambia el nombre.

**Why this priority**: el nombre es la referencia legible que usa el admin para distinguir
promociones al listar y al resolver solapamientos — un duplicado silencioso rompería esa
referencia.

**Independent Test**: se puede probar creando dos promociones con el mismo `name` y verificando
el 409 en el segundo intento, en cada uno de los tres endpoints, sin pasar por un cobro real.

**Acceptance Scenarios**:

1. **Given** una promoción existente `name="Martes de granizado"`, **When** se pide `POST
   /promotions` con el mismo `name`, **Then** 409 "Ya existe una promoción con ese nombre"
   (`RN-PROMO-78`).
2. **Given** dos promociones existentes A y B, **When** se pide `PATCH /promotions/{A.id}
   {"name":"<nombre de B>"}`, **Then** 409 — el router valida unicidad excluyendo el propio
   `A.id` antes de invocar `service.update` (`RN-PROMO-78`, evidencia: `router.py:72-74`).
3. **Given** una promoción A, **When** se pide `PATCH /promotions/{A.id}` sin tocar `name` (no
   presente en el payload), **Then** el chequeo de unicidad se omite por completo — no se
   ejecuta ninguna consulta de unicidad porque `"name" not in body.model_fields_set`
   (`router.py:72`).
4. **Given** una promoción A, **When** se pide `POST /promotions/{A.id}/duplicate` con un
   `name` ya usado por otra promoción, **Then** 409 antes de llamar a `service.duplicate`
   (`RN-PROMO-59`, `RN-PROMO-78`).

---

### User Story 6 - El solapamiento entre promociones es solo advertencia, nunca bloquea create/update (Priority: P2) — regla compartida, referenciada desde la spec 012

`find_overlaps` se invoca después de cada `create`/`update`/`update_shape` exitoso y se anexa a
la respuesta como campo `overlaps`, pero su resultado nunca aborta la operación — el mecanismo
de detección en sí (`_ranges_overlap`, `_csv_overlap`, `_times_overlap`, `_scope_overlap`) ya se
documenta en la spec 012 (User Story 6), porque vive en `promotions/service.py` y es compartido
por ambos dominios (cálculo y administración). Esta spec solo confirma que, desde el lado de la
administración, el resultado es puramente informativo.

**Why this priority**: es el mecanismo que hace posible que dos promociones legítimamente
superpuestas (por ejemplo, "10% en granizados" y "20% los martes") coexistan sin que crear la
segunda bloquee la primera — el propio comentario del router lo explica: un bloqueo duro haría
imposibles sus propios casos de uso.

**Independent Test**: se puede probar creando o actualizando dos promociones con rangos de
fecha, horario o alcance que se cruzan deliberadamente, verificando que ambas operaciones
devuelven 201/200 con un campo `overlaps` no vacío, sin que ninguna sea rechazada.

**Acceptance Scenarios**:

1. **Given** una promoción vigente "10% en granizados" (categoría), **When** se crea "20% los
   martes en Granizado de mora" (producto, dentro de esa categoría, mismo rango horario), **Then**
   la creación se acepta con 201, y la respuesta incluye a la primera en su lista `overlaps`
   (`RN-PROMO-30`, `RN-PROMO-60`).
2. **Given** el resultado de `find_overlaps` en cualquier `create`/`update`/`update_shape`,
   **When** se filtran los candidatos considerados, **Then** solo se comparan promociones
   `status` en `{draft, active, paused}` y `type` en `AUTO_TYPES` (`percent`, `fixed`,
   `qty_price`) — `finished` y `combo` quedan siempre excluidos del chequeo de solapamiento
   (`RN-PROMO-61`).

---

### User Story 7 - A-30: dos vectores de `IntegrityError` no controlado en `PATCH` (Priority: P1) — hallazgo con impacto real, primer candidato a test explícito

Dos casos donde el `PATCH` escalar puede llegar a `commit()` con datos que violan una
restricción de base de datos, sin que ninguna capa de aplicación lo intercepte antes: (1)
`PATCH {"name": null}` — `PromotionUpdate.name` es `Optional[str]` con `min_length=1`, pero
Pydantic no aplica `min_length` cuando el valor es literalmente `None`; el router solo valida
unicidad "si `name is not None`" (User Story 5, escenario 3); el servicio copia cualquier campo
presente en `model_fields_set`, incluido `name=None`; la columna `name` es `nullable=False`. (2)
Dos `targets` repetidos (mismo `product_id` o `category_id` dentro de la misma promoción) — a
diferencia de `combo_items` (validado explícitamente por `RN-PROMO-72`) o del `name` (validado
por `ensure_unique`), los `targets` duplicados solo están protegidos por un índice único parcial
de PostgreSQL (`uq_promotion_targets_product`, `uq_promotion_targets_category`), sin ningún
validador de Pydantic ni de servicio que los intercepte antes de `db.flush()`.

**Why this priority**: a diferencia de los demás hallazgos de esta spec, este es un **500 real
no controlado**, no solo un caso límite de configuración — cualquier admin puede dispararlo por
accidente desde el panel, y el resultado es un error genérico sin mensaje de negocio útil.

**Independent Test**: (1) se puede probar con `PATCH /promotions/{id} {"name": null}` sobre
cualquier promoción existente, verificando que Pydantic acepta el payload y que el fallo real
ocurre recién en `commit()`. (2) se puede probar con `PATCH /shape` enviando dos `targets` con
el mismo `product_id`, verificando que la validación de Pydantic y `_apply_targets` los aceptan
y que el fallo ocurre en `db.flush()`.

**Acceptance Scenarios**:

1. **Given** una promoción existente cualquiera, **When** se pide `PATCH /promotions/{id}
   {"name": null}`, **Then** el payload pasa la validación de Pydantic (`min_length=1` no se
   evalúa sobre `None`) y pasa el chequeo de unicidad del router (`body.name is not None` es
   falso, así que la validación de `ensure_unique` se omite por completo), y el sistema llega a
   `promo.name = None` seguido de `db.flush()`/`commit()` contra una columna `NOT NULL` —
   comportamiento observado: `IntegrityError` no controlado (`RN-PROMO-75`).
2. **Given** una promoción cualquiera en `draft`, **When** se pide `PATCH /shape` con dos
   `targets` que apuntan al mismo `product_id`, **Then** `TargetIn._one_scope` (XOR por target
   individual) no detecta el problema porque evalúa cada target por separado, y `_apply_targets`
   inserta ambas filas sin comparar contra las existentes — el índice único parcial de
   PostgreSQL es la única barrera, y su violación no tiene traducción a un mensaje de negocio en
   esta capa (`RN-PROMO-76`).

**Tratamiento acordado** (`registro-de-anomalias.md`, A-30): el vector (1) es **ACCIDENTAL**
(`RN-PROMO-75`) — el propio código ya documenta haber corregido un bug equivalente para la
unicidad del nombre completo (`router.py:70-71`, comentario explícito), pero no cubrió el caso
`name=null`, dejando un vector del mismo patrón sin resolver. El vector (2) es **PENDIENTE**
(`RN-PROMO-76`) — depende de si existe algún manejo genérico de `IntegrityError` fuera de este
módulo (`app/main.py` u otro middleware) que convierta la violación del índice en un 409
legible; no verificado en este reconocimiento. **Corregir en fase de modernización**: validar
explícitamente ambos casos en la capa de servicio, con mensajes de negocio dedicados. Sin riesgo
retroactivo — es lógica de validación, no dato ya almacenado.

---

### User Story 8 - A-37 (porción administración): reenviar el mismo estado es un no-op silencioso; una promoción puede nacer directamente `active`/`paused` (Priority: P3) — `PENDIENTE`, sin especificar

Dos comportamientos de la máquina de estados, de los siete del cluster completo de A-37 (los
otros cinco pertenecen al cálculo de descuento, spec 012): (1) `change_status` compara
`new_status == promo.status` **antes** de consultar `PROMOTION_TRANSITIONS`, devolviendo la
promoción sin cambios si el estado solicitado es el mismo que el actual — incluido el caso
`finished→finished`, que de otro modo sería rechazado porque `finished` no tiene transiciones
salientes; (2) `PromotionCreate` solo prohíbe expresamente `status=finished` al crear
(`_status_on_create`); `status=active` o `status=paused` se aceptan directamente en el `POST`,
sin exigir que la promoción pase primero por `draft`.

**Why this priority**: son hallazgos de diseño de la máquina de estados, sin impacto económico
demostrado — comparten prioridad P3 con el resto del cluster A-37 ya documentado en la spec 012.

**Independent Test**: (1) se puede probar invocando `change_status(db, promo, promo.status)`
sobre una promoción `finished`, verificando que devuelve 200 sin excepción. (2) se puede probar
con `POST /promotions {"status":"active", ...}`, verificando que se acepta con 201.

**Acceptance Scenarios**:

1. **Given** una promoción `status="finished"`, **When** se pide `PATCH /status
   {"status":"finished"}` (el mismo estado actual), **Then** el sistema responde 200 sin
   error — el chequeo `new_status == promo.status` intercepta el pedido antes de consultar la
   tabla de transiciones, donde `finished` no tiene ningún destino permitido (`RN-PROMO-41`,
   `RN-PROMO-54`).
2. **Given** cualquier promoción en cualquier estado, **When** se reenvía su propio estado
   actual vía `PATCH /status`, **Then** el resultado es siempre un no-op exitoso — el mismo
   patrón aplica a `draft→draft`, `active→active` y `paused→paused` (`RN-PROMO-54`).
3. **Given** `POST /promotions {"status":"active", "type":"percent", "value":10, ...}`, **When**
   se valida, **Then** se acepta con 201 — la promoción nace directamente `active`, sin haber
   pasado nunca por `draft` (`RN-PROMO-68`).
4. **Given** `POST /promotions {"status":"finished", ...}`, **When** se valida, **Then** 422
   "Una promoción no puede crearse finalizada" — este es el único estado de creación
   explícitamente prohibido (`RN-PROMO-68`).

**Tratamiento acordado** (`registro-de-anomalias.md`, A-37, porción administración):
**PENDIENTE** ambos, sin impacto económico — comparten clasificación con las cinco porciones de
cálculo ya documentadas en la spec 012. Documentar sin especificar hasta respuesta de negocio.
Preguntas abiertas asociadas: ¿la intención es que reconfirmar un estado sea siempre un no-op
exitoso, o debería rechazarse igual que cualquier transición no listada? ¿El negocio pretendía
que TODA promoción pasara obligatoriamente por `draft` antes de `active`? (`reglas-de-negocio.md`,
preguntas abiertas #35, #36 referenciadas por A-37).

---

### User Story 9 - A-39 (porción administración): el job de medianoche compara vigencia en UTC absoluto, distinto del criterio real (Priority: P3) — `ACCIDENTAL`, sin riesgo económico

`expire_promotions` (job `CronTrigger(hour=0, minute=0)`, id `expire_promotions`) marca en masa
`status="finished"` toda promoción `active`/`paused`/`draft` cuyo `ends_at` sea anterior a
`now=datetime.now(timezone.utc)` sin conversión a hora local y comparado como datetime completo
(no solo por fecha). Esto es un criterio de corte distinto al de `_valid_now` (spec 012:
hora local del tenant, `ends_at` comparado solo por fecha) — el job puede marcar `finished` una
promoción que la evaluación real en el momento de cobrar seguiría considerando vigente, con un
desfase de hasta 1.5 días en UTC-5. El propio comentario del job se autodescribe "puramente
informativo": `_valid_now` es la autoridad real en cada evaluación, y el job solo mantiene
`status` como una etiqueta administrativa. Corre por tenant con un lock distribuido en Redis
(mismo patrón que el barrido de sesiones de mesa, spec 010) — un tenant sin Redis disponible
simplemente omite el ciclo (`RN-SCHED-08`, fuera de alcance detallado aquí).

Una promoción `draft` con `ends_at` ya vencido pasa directo a `finished` sin haber estado nunca
`active` — el job no distingue "nunca se activó" de "estuvo activa y venció" (`RN-SCHED-10`).

**Why this priority**: el propio job se documenta como no autoritativo — el monto que se cobra
realmente nunca se ve afectado (`_valid_now` sigue gobernándolo, spec 012), solo la etiqueta de
estado `finished` que ve el administrador en el listado. P3, comparable en impacto a los demás
hallazgos de bajo riesgo de esta spec.

**Independent Test**: se puede verificar por inspección de código, comparando la construcción de
`now` en `core/scheduler.py:224` (`datetime.now(timezone.utc).replace(tzinfo=None)`, UTC crudo,
datetime completo) contra `_valid_now`/`local_now` en `promotions/service.py:57-68,91-99` (hora
local del tenant, `ends_at` solo por fecha), sin necesitar ejecutar el job real con datos de
producción.

**Acceptance Scenarios**:

1. **Given** un tenant en `America/Bogota` (UTC-5) y una promoción `status="active"`,
   `ends_at="2026-08-04T00:00:00"`, **When** el reloj real marca `2026-08-03 19:00` hora local
   (`2026-08-04T00:00:00` UTC), **Then** el job `expire_promotions` ya la marca `finished`
   (`ends_at < now` es cierto en UTC absoluto), pero `_valid_now` la seguiría considerando
   vigente hasta las 23:59:59 locales del 4 de agosto — desfase de casi un día y medio
   (`RN-SCHED-11`).
2. **Given** una promoción `status="draft"` con `ends_at` ya vencido (nunca llegó a activarse),
   **When** corre el job de medianoche, **Then** pasa directo a `status="finished"` — el job
   filtra `status IN (active, paused, draft)`, sin excluir `draft` (`RN-SCHED-10`).
3. **Given** una promoción marcada `finished` por el job pese a que `_valid_now` la seguiría
   considerando vigente (escenario 1), **When** se evalúa un cobro real en ese instante, **Then**
   el monto cobrado NO se ve afectado por la etiqueta `finished` — la spec 012 documenta que
   `_valid_now` filtra por `status IN ('active')` en el motor automático, así que en la práctica
   esta promoción específica sí dejaría de aplicar apenas el job la marca `finished`, **incluso
   si `_valid_now` (evaluado de forma aislada, sin el filtro de `status`) seguiría considerando
   vigentes su fecha/hora/día** — la discrepancia real y verificable es entre los dos *criterios
   de corte de fecha*, no entre dos resultados de cobro observables de forma independiente
   (matiz de precisión sobre `RN-SCHED-11`, ver Assumptions).

**Tratamiento acordado** (`registro-de-anomalias.md`, A-39): **ACCIDENTAL** — contradicción
directa y verificable en código entre los dos criterios de corte de fecha, aunque el propio
comentario del job se autodescribe como "puramente informativo". **Corregir en fase de
modernización**, unificando el criterio de corte del job con `local_now()`/`_valid_now`. Sin
riesgo retroactivo — es lógica de un job de mantenimiento, no dato transaccional. Pregunta de
negocio abierta: ¿debe unificarse el criterio, o el desfase es aceptable porque el job es
"puramente informativo"? (`reglas-de-negocio.md`, pregunta abierta #38).

---

### Edge Cases

- **`PATCH /status` reenviando el estado actual, incluido `finished→finished`**: no-op
  silencioso, no pasa por la tabla de transiciones (`RN-PROMO-54`, User Story 8).
- **`PATCH /shape` sobre una promoción que no está en `draft`**: rechazado con 409 sin importar
  cuál de los tres campos de forma (`type`, `targets`, `combo_items`) se intente cambiar
  (`RN-PROMO-56`, User Story 2).
- **`PATCH {"name": null}`**: pasa Pydantic y el chequeo de unicidad del router, llega a
  `commit()` contra una columna `NOT NULL` — vector de `IntegrityError` no controlado
  (`RN-PROMO-75`, User Story 7, A-30).
- **`PATCH /shape` con dos `targets` del mismo `product_id`/`category_id`**: solo protegido por
  un índice único parcial en BD, sin validador de aplicación — mismo patrón de riesgo que el
  caso anterior (`RN-PROMO-76`, User Story 7, A-30).
- **Combo con un componente borrado en cascada después de haber sido creado con `>=2`**:
  revalidado al intentar `PATCH /status {"status":"active"}`, no solo en la creación
  (`RN-PROMO-55`, User Story 1).
- **`POST /promotions` con `status="active"` o `status="paused"` directo**: aceptado, sin pasar
  por `draft` — solo `status="finished"` está prohibido en la creación (`RN-PROMO-68`, User
  Story 8).
- **Dos promociones con rangos/horarios que se solapan**: nunca bloquea create/update, solo se
  informa en el campo `overlaps` de la respuesta (`RN-PROMO-30`, `RN-PROMO-60`, User Story 6).
- **Promoción `draft` con `ends_at` ya vencido, nunca activada**: el job de medianoche la marca
  `finished` igual que si hubiera estado `active` (`RN-SCHED-10`, User Story 9).

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001 [Regla crítica]**: Solo una promoción con `status="active"` DEBE habilitar el
  descuento automático; los demás estados (`draft`, `paused`, `finished`) NUNCA lo habilitan,
  sin importar si la fecha/hora/día serían vigentes (`RN-PROMO-46`).
- **FR-002**: La máquina de estados DEBE tener transiciones fijas: `draft→{active, finished}`,
  `active→{paused, finished}`, `paused→{active, finished}`, `finished→{}` (terminal, sin
  transiciones salientes). Cualquier transición fuera de este mapa DEBE rechazarse con 409
  (`RN-PROMO-53`).
- **FR-003**: Al transicionar un combo a `status="active"`, el sistema DEBE revalidar que tenga
  al menos 2 `product_variant_id` distintos en `combo_items` — defensa contra el borrado en
  cascada de una variante después de la creación (`RN-PROMO-55`).
- **FR-004 [Regla crítica]**: El cambio de `type`, `targets` o `combo_items` de una promoción
  existente SOLO DEBE permitirse mientras su `status` sea `draft`; cualquier intento en otro
  estado DEBE rechazarse con 409 (`RN-PROMO-40`, `RN-PROMO-56`).
- **FR-005**: El endpoint de actualización escalar (`PATCH /promotions/{id}`) NO DEBE aceptar
  `type`, `targets` ni `combo_items` en su schema — esos campos, si se envían, se descartan
  silenciosamente sin generar error; solo `PATCH /promotions/{id}/shape` los modifica
  (`RN-PROMO-57`).
- **FR-006**: Duplicar una promoción DEBE crear siempre la copia con `status="draft"`, sin
  importar el estado de la promoción de origen, copiando todos sus campos escalares, `targets`
  y `combo_items` (`RN-PROMO-42`, `RN-PROMO-58`).
- **FR-007**: Duplicar DEBE exigir un `name` nuevo y único, validado antes de ejecutar la copia
  (`RN-PROMO-59`).
- **FR-008 [Regla crítica, triple capa]**: Un descuento `percent` NUNCA DEBE superar `value=100`
  — validado en el schema de creación, revalidado en el servicio de actualización (porque
  `PromotionUpdate` no conoce el `type` y no puede autovalidarse), y protegido por un `CHECK` de
  base de datos como última red (`RN-PROMO-62`).
- **FR-009 [Regla crítica, triple capa]**: Una promoción `qty_price` DEBE exigir `min_qty>=2` a
  nivel de la promoción, con el mismo patrón de triple capa que `FR-008` (`RN-PROMO-63`).
- **FR-010**: El `min_qty` de un `target` individual, cuando se define, también DEBE ser `>=2`
  (`RN-PROMO-64`).
- **FR-011**: `priority` DEBE estar acotado al rango `[0, 1000]` inclusive, tanto en creación
  como en actualización (`RN-PROMO-73`).
- **FR-012**: `start_time` y `end_time` DEBEN configurarse juntos (ambos presentes o ambos
  ausentes) — validado en el payload de entrada y revalidado en el servicio contra el estado
  final tras un `PATCH`, para detectar el caso de limpiar solo uno de los dos a `null`
  (`RN-PROMO-65`).
- **FR-013**: `ends_at` NO DEBE tener fecha anterior a `starts_at`, comparados por fecha (no por
  datetime completo) (`RN-PROMO-66`).
- **FR-014**: `days_of_week` DEBE aceptarse solo como CSV de enteros en `[0,6]`; el sistema DEBE
  normalizarlo (deduplicado y ordenado) antes de guardarlo (`RN-PROMO-67`).
- **FR-015 [`[DUDOSA]`, anomalía A-37, `PENDIENTE` — documentada sin especificar como contrato]**:
  Una promoción NO DEBE poder crearse con `status="finished"`, pero SÍ DEBE poder crearse
  directamente con `status="active"` o `status="paused"`, sin exigir que pase primero por
  `draft` (`RN-PROMO-68`).
- **FR-016**: El precio/tamaño de paquete de un `target` (`value`, `min_qty` propios del target)
  SOLO DEBE aceptarse cuando `type="qty_price"` — tanto en la creación como en un cambio de
  forma; en cualquier otro tipo, DEBE rechazarse con 422 (`RN-PROMO-69`).
- **FR-017**: Una promoción `qty_price` DEBE exigir al menos un `target`, y TODOS sus `targets`
  DEBEN tener `value` y `min_qty` definidos — sin excepción, tanto en creación como en cambio de
  forma (`RN-PROMO-70`).
- **FR-018 [Regla crítica]**: `TargetIn` DEBE exigir exactamente uno de `product_id` o
  `category_id` (XOR estricto a nivel de aplicación); el `CHECK` de base de datos es más laxo y
  solo exige "al menos uno" — la exclusividad depende enteramente de la capa de aplicación
  (`RN-PROMO-71`).
- **FR-019 [Regla crítica]**: Un combo DEBE exigir al menos 2 `product_variant_id` distintos en
  `combo_items`, sin duplicados, y NO DEBE aceptar `targets` a la vez — el alcance de un combo se
  define exclusivamente en `combo_items` (`RN-PROMO-72`).
- **FR-020**: `value` (monto o porcentaje del descuento) NUNCA DEBE ser negativo, tanto a nivel
  de la promoción como del target — validado en schema y protegido por `CHECK` de base de datos
  (`RN-PROMO-74`).
- **FR-021 [Regla crítica]**: El nombre de una promoción DEBE ser único, validado en los tres
  puntos de entrada que pueden fijarlo: creación, actualización (solo si `name` está presente y
  no es `null`) y duplicado (`RN-PROMO-78`).
- **FR-022 [`[ACCIDENTAL]`, anomalía A-30, hallazgo con impacto real — corregir en
  modernización]**: `PATCH {"name": null}` DEBE producir hoy un `IntegrityError` no controlado:
  pasa la validación de Pydantic (`min_length` no se aplica sobre `None`), pasa el chequeo de
  unicidad del router (que solo valida "si `name is not None`"), y llega a `commit()` contra una
  columna `name` `NOT NULL` (`RN-PROMO-75`).
- **FR-023 [`[DUDOSA]`, anomalía A-30, `PENDIENTE` — corregir en modernización]**: Dos `targets`
  repetidos (mismo `product_id` o `category_id` en la misma promoción) DEBEN producir hoy un
  fallo a nivel de índice único parcial de PostgreSQL — no existe ningún validador de Pydantic
  ni de servicio que los intercepte antes; se desconoce si algún manejo genérico de
  `IntegrityError` fuera de este módulo lo convierte en un error legible (`RN-PROMO-76`).
- **FR-024**: `min_qty` a nivel de la promoción DEBE tener un mínimo de 1, tanto en creación
  como en actualización (`RN-PROMO-77`).
- **FR-025**: El solapamiento detectado entre dos promociones (mecanismo documentado en la spec
  012) DEBE ser exclusivamente informativo también desde el lado de la administración — NUNCA
  DEBE bloquear `create`, `update` ni `update_shape` (`RN-PROMO-30`, `RN-PROMO-60`).
- **FR-026**: El chequeo de solapamiento invocado tras `create`/`update`/`update_shape` SOLO DEBE
  considerar como candidatos promociones con `status` en `{draft, active, paused}` y `type` en
  `AUTO_TYPES` (`percent`, `fixed`, `qty_price`) — `finished` y `combo` quedan siempre excluidos
  (`RN-PROMO-61`).
- **FR-027 [`[DUDOSA]`, anomalía A-37, `PENDIENTE` — documentada sin especificar como contrato]**:
  Reenviar el mismo estado actual de una promoción vía `PATCH /status` (incluido
  `finished→finished`) DEBE producir un no-op silencioso con 200 — el chequeo de igualdad ocurre
  antes de consultar la tabla de transiciones, sin pasar por ella (`RN-PROMO-41`, `RN-PROMO-54`).
- **FR-028 [`[ACCIDENTAL]`, anomalía A-39, corregir en modernización]**: El job periódico de
  medianoche (`expire_promotions`, `CronTrigger(hour=0, minute=0)`) DEBE marcar `status="finished"`
  toda promoción `active`/`paused`/`draft` cuyo `ends_at` sea anterior a
  `now=datetime.now(timezone.utc)` sin conversión a hora local y comparado como datetime completo
  — un criterio de corte distinto del que usa `_valid_now` (hora local del tenant, `ends_at`
  comparado solo por fecha, spec 012), pudiendo generar un desfase de hasta 1.5 días en UTC-5
  entre cuándo el job etiqueta `finished` y cuándo la evaluación real dejaría de considerar
  vigente la promoción (`RN-SCHED-11`).
- **FR-029**: El job de medianoche DEBE incluir `status="draft"` entre los estados que puede
  marcar `finished` por vencimiento — una promoción que nunca se activó y cuyo `ends_at` ya pasó
  también se marca `finished`, sin distinguir "nunca se activó" de "estuvo activa y venció"
  (`RN-SCHED-10`).
- **FR-030**: El job de medianoche DEBE correr por tenant con un lock distribuido en Redis
  (mismo mecanismo que el barrido de sesiones de mesa, spec 010); si Redis no está disponible
  para un ciclo, el job DEBE omitirse silenciosamente ese ciclo sin bloquear el arranque de la
  aplicación (`RN-SCHED-08`, fuera de alcance detallado de esta spec).

### Key Entities *(include if feature involves data)*

- **Promotion**: la promoción o combo administrado por esta spec. Atributos relevantes:
  `status` (máquina de estados, `FR-001`-`FR-003`), `type`/`targets`/`combo_items` (forma,
  cambiable solo en `draft`, `FR-004`-`FR-005`), `name` (único, `FR-021`-`FR-022`), `value`
  (tope porcentual, `FR-008`), `priority` (rango `FR-011`), `starts_at`/`ends_at`/`days_of_week`/
  `start_time`/`end_time` (vigencia configurada, `FR-012`-`FR-014`).
- **PromotionTarget**: el alcance de una promoción sobre un producto o categoría específicos.
  Atributos relevantes a esta spec: `product_id` XOR `category_id` (`FR-018`), unicidad no
  validada en aplicación (`FR-023`), `value`/`min_qty` propios (solo `qty_price`,
  `FR-016`-`FR-017`).
- **PromotionComboItem**: componente de la receta de un combo (`combo_id`, variante requerida,
  cantidad). Gobierna la exigencia mínima de 2 componentes distintos, tanto en creación como en
  la transición a `active` (`FR-003`, `FR-019`).
- **PROMOTION_TRANSITIONS**: mapa de transiciones fijas por estado origen (`FR-002`), definido
  en el modelo (`app/models/promotion.py`), consumido por `change_status`.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: El 100% de las reglas `RN-PROMO-46` a `RN-PROMO-78` y `RN-SCHED-10`/`RN-SCHED-11`
  puede verificarse ejecutando los pasos descritos en esta spec contra un `pos-backend` en
  ejecución, sin necesitar leer `router.py`/`schemas.py`/`service.py`/`scheduler.py` para
  entender el comportamiento esperado.
- **SC-002**: La máquina de estados (`FR-002`) queda fijada como invariante de test obligatorio
  en cualquier reimplementación — ningún cambio futuro puede introducir una transición fuera del
  mapa `PROMOTION_TRANSITIONS` sin que un test de caracterización dedicado lo detecte. Hoy
  ningún script de los 12 disponibles (`test_promotions_rules.py` incluido) cubre la máquina de
  estados ni la validación de forma del `PATCH`/`PATCH /shape` de forma dedicada — es un gap de
  caracterización explícito de esta spec.
- **SC-003**: A-30 (User Story 7) queda documentado con evidencia suficiente para escribir el
  primer test explícito de esta spec — es el único hallazgo del alcance que produce un 500 real
  no controlado, no solo un caso límite de configuración.
- **SC-004**: La regla de triple capa del tope porcentual (`FR-008`) y de `qty_price`
  (`FR-009`) quedan fijadas como invariante — cualquier reimplementación que remueva una de las
  tres capas (schema, servicio o `CHECK` de BD) sigue estando protegida por las otras dos, y un
  test de caracterización que ejercite las tres por separado lo confirma.
- **SC-005**: Las anomalías con clasificación cerrada de esta spec quedan documentadas con su
  tratamiento acordado (A-30 vector 1 `ACCIDENTAL`, A-39 `ACCIDENTAL`, ambas "corregir en
  modernización sin riesgo retroactivo"), de forma que la modernización no las reintroduzca por
  accidente.
- **SC-006**: Las anomalías sin decisión de negocio (A-30 vector 2, A-37 porción administración)
  quedan documentadas con su comportamiento observado y su evidencia de código, sin fijarse como
  contrato deseado, para que la próxima ronda de entrevista de negocio las resuelva con contexto
  completo sin necesidad de releer el código fuente.

## Out of Scope

- **Cómo se calcula el descuento en tiempo real** una vez la promoción ya existe y está
  `active` (vigencia hora local, selección de mejor promoción por línea, cálculo por tipo,
  expansión de combo) — cubierto por la spec 012; esta spec documenta cómo esa promoción llegó a
  existir y a tener la forma que tiene, no cómo se usa para cobrar.
- **El consumo de las promociones administradas aquí en el menú público y el carrito del
  comensal por QR** — cubierto por la spec 007.
- **El consumo de estas promociones en cada camino de cobro real** (mostrador, cierre unificado,
  cierre dividido de mesa, `pay_order` legado) — cubierto por las specs 008, 010 y 011.
- **El panel de administración de promociones del frontend** (`promotions-page.component.ts`,
  formularios de creación/edición, presentación de `overlaps`) — el mecanismo de backend que esa
  UI consume se documenta aquí; su presentación visual no es objeto de esta spec.
- **El resto del cluster A-37** (las cinco porciones de cálculo de descuento: `qty_price`/combo
  sin descuento negativo, descuento `0` como descalificador, truncamiento de cantidad, combo sin
  aviso al vencer) — cubierto por la spec 012, User Story 10.
- **El resto del cluster A-36** (precisión de `starts_at`/`ends_at`, solapamiento de horario
  incompleto, ventana con cruce de medianoche sin test en su segundo límite) — cubierto por la
  spec 012, User Story 12; ninguna porción de A-36 pertenece al dominio de administración.
- **El mecanismo de detección de solapamiento en sí** (`find_overlaps`, `_ranges_overlap`,
  `_csv_overlap`, `_times_overlap`, `_scope_overlap`) — documentado en detalle en la spec 012,
  User Story 6, porque vive en `promotions/service.py` compartido por ambos dominios; esta spec
  solo confirma su carácter informativo desde el lado de la administración (User Story 6).

## Assumptions

- **Esta es una spec de ingeniería inversa, no de una feature nueva**: a diferencia del resto de
  las guías de este template ("evitar detalles de implementación"), aquí los nombres de función,
  constantes internas (`PROMOTION_TRANSITIONS`, `AUTO_TYPES`), estados y valores literales
  **son** el contrato observable que se está documentando — se citan explícitamente porque los
  criterios de aceptación deben ser verificables directamente contra `pos-backend` en ejecución.
- **A-30 se documenta con dos vectores de clasificación distinta**: el vector de `name=null`
  (`RN-PROMO-75`) es `ACCIDENTAL` porque hay evidencia directa en el propio código de que un bug
  equivalente (unicidad completa del nombre) ya fue corregido, dejando este caso específico sin
  cubrir por el mismo patrón de corrección. El vector de `targets` duplicados (`RN-PROMO-76`) es
  `PENDIENTE` porque depende de un hecho no verificado en este reconocimiento (si existe manejo
  genérico de `IntegrityError` en `app/main.py` u otro middleware) — no se asume ni se descarta
  su existencia.
- **A-37 (porción administración: `RN-PROMO-41`/`RN-PROMO-54`, `RN-PROMO-68`) se documenta pero
  NO se especifica como contrato**: siguiendo instrucción explícita de alcance, queda con
  clasificación `PENDIENTE` en `registro-de-anomalias.md` — se describe el comportamiento
  observado hoy, pero no se fija como comportamiento correcto ni obligatorio para la
  modernización.
- **A-39 se documenta como discrepancia `ACCIDENTAL` sin riesgo económico confirmado**: el propio
  comentario del código autodescribe el job como "puramente informativo"; esta spec no asume que
  esa autodescripción sea completa — la User Story 9 deja constancia del matiz de que, en la
  práctica, la etiqueta `finished` sí puede afectar si una promoción sigue apareciendo como
  candidata en el filtro SQL parcial de vigencia (`status='active'`) que usa el motor automático
  (spec 012, `RN-PROMO-17`), aunque el *monto calculado* para una promoción que sigue `active`
  nunca cambia por causa de este job.
- **`test_promotions_rules.py` (único script que corre en CI) no cubre ningún escenario de esta
  spec**: se verificó por inspección que su cobertura se limita a `_valid_now`/`_in_time_window`
  (vigencia, compartida conceptualmente con la spec 012) — ningún `check()` del script ejercita
  `change_status`, `update_shape`, `duplicate` ni las validaciones de `schemas.py`. Esto se
  documenta como gap de caracterización explícito (`SC-002`), no como ausencia de comportamiento.
- **Los números de línea citados como evidencia corresponden al estado del código en la fecha de
  creación de esta spec (2026-08-16)**: cualquier refactor posterior (aunque esta fase prohíbe
  introducirlos, Principio IV de la Constitución) invalidaría las referencias exactas de línea,
  no el enunciado de la regla en sí.
