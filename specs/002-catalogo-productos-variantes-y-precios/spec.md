# Feature Specification: Catálogo de productos, variantes y precios

**Feature Branch**: `002-catalogo-productos-variantes-y-precios`

**Created**: 2026-08-16

**Status**: Draft

**Naturaleza de esta spec**: **ingeniería inversa / characterization spec**. No describe una
funcionalidad nueva: documenta el comportamiento que el sistema **ya tiene hoy** en
`pos-backend/app/api/v1/catalog/service.py`, `app/api/v1/catalog/router.py` y
`app/api/v1/products/`, para que sirva de contrato formal de cara a la modernización
(Principio I y Principio III de la [Constitución](../../.specify/memory/constitution.md)). Donde
el resto de las specs de este proyecto describen lo que el sistema **debe** hacer, esta describe
lo que el sistema **efectivamente hace** — incluida su única anomalía conocida en este recorte
(A-44), con su tratamiento acordado citado de `registro-de-anomalias.md`.

**Input**: User description: "Spec de ingeniería inversa: documenta el comportamiento EXISTENTE
del catálogo de productos, variantes y precios del sistema POS Heladería, tomado de
`reglas-de-negocio.md` (RN-CAT-01 a RN-CAT-11) y de `registro-de-anomalias.md` (A-44), para que
sirva de contrato en la modernización."

## User Scenarios & Testing *(mandatory)*

<!--
  Cada escenario documenta un comportamiento OBSERVADO en `catalog/service.py`, `catalog/router.py`
  y `products/service.py`, no uno deseado. La anomalía conocida se marca inline con su
  tratamiento acordado (registro-de-anomalias.md).
-->

### User Story 1 - Precio de una línea vendida (Priority: P1)

Un cajero agrega al carrito una presentación de un producto (p. ej. «Copa Grande») con algunos
sabores o toppings que tienen recargo. El sistema calcula el precio de esa línea sumando el
precio de la presentación y el recargo de cada opción elegida, sin aplicar redondeo, descuentos
ni impuestos.

**Why this priority**: es el cálculo que determina cuánto se cobra en cada venta — cualquier
divergencia aquí afecta directamente la caja.

**Independent Test**: se puede probar de forma aislada invocando `compute_line_price(variant,
options)` con una variante y una lista de opciones, sin depender de carrito, inventario ni caja.

**Acceptance Scenarios**:

1. **Given** una variante «Copa Grande» con `price=15000` y las opciones «Chocolate»
   (`extra_price=1000`) y «Maní» (`extra_price=500`), **When** se calcula el precio de la línea,
   **Then** el resultado es `15000 + 1000 + 500 = 16500` (`RN-CAT-01`).
2. **Given** el mismo cálculo, **When** se inspecciona el resultado, **Then** es la suma
   aritmética exacta de los `Decimal` de origen — no se aplica ninguna función de redondeo o
   truncamiento (`quantize`/`round`) antes de devolver el precio de línea. La columna `Numeric(12,
   2)` en base de datos es el único límite de precisión, y actúa solo al persistir la variante u
   opción, no en este cálculo (`RN-CAT-02`).
3. **Given** una variante sin ninguna opción elegida, **When** se calcula el precio de línea,
   **Then** el resultado es exactamente el precio de la variante, sin ningún ajuste adicional
   (`RN-CAT-01`).

---

### User Story 2 - Precios no negativos, pero sí en cero (Priority: P1)

Un administrador crea o edita una presentación o una opción. El sistema permite dejarla a precio
0 (cortesías, productos aún sin tarifa, sabores sin recargo) pero nunca acepta un precio
negativo.

**Why this priority**: es la validación que impide que un error de captura convierta una venta en
una salida de dinero de caja.

**Independent Test**: se puede probar enviando `POST /products/{id}/variants` y
`POST /option-groups/{id}/options` con `price`/`extra_price` en `-1`, `0` y un valor positivo,
sin depender de ningún otro módulo.

**Acceptance Scenarios**:

1. **Given** `VariantCreate(price=-1)`, **When** se envía a `POST /products/{id}/variants`,
   **Then** el schema rechaza la petición con `422` antes de tocar base de datos (`RN-CAT-03`).
2. **Given** `VariantCreate(price=0)`, **When** se envía a `POST /products/{id}/variants`,
   **Then** la variante se crea con éxito — precio 0 es un valor válido, no un error
   (`RN-CAT-03`). La misma variante también tiene, como respaldo redundante, el `CHECK
   ck_product_variant_price_positive (price >= 0)` en base de datos.
3. **Given** `OptionCreate(name="Vainilla")` sin especificar `extra_price`, **When** se envía a
   `POST /option-groups/{id}/options`, **Then** la opción se guarda con `extra_price=0` por
   defecto (`RN-CAT-04`), respaldado por el `CHECK ck_option_extra_price_positive (extra_price >=
   0)` en base de datos.

---

### User Story 3 - Todo producto nuevo nace vendible (Priority: P1)

Un administrador da de alta un producto nuevo en el catálogo (p. ej. «Cono Waffle») sin definir
todavía sus presentaciones. El sistema le crea automáticamente una presentación «Single» a precio
0 para que el producto sea vendible de inmediato, sin dejarlo en un estado intermedio sin nada
que cobrar.

**Why this priority**: sin esto, un producto recién creado no tendría ninguna línea vendible y no
podría agregarse a una venta hasta que alguien recuerde crear al menos una variante.

**Independent Test**: se puede probar completamente creando un producto vía `POST /products` y
verificando que trae exactamente una variante «Single» con precio 0, sin depender de ningún otro
módulo.

**Acceptance Scenarios**:

1. **Given** los datos de un producto nuevo sin variantes, **When** `POST /products`, **Then** el
   sistema crea el producto y, en la misma transacción, le agrega una variante `name="Single"`,
   `price=0`, `active=True` con SKU autogenerado (`ensure_default_variant`, `RN-CAT-05`).
2. **Given** un producto que ya tiene al menos una variante, **When** se invoca
   `ensure_default_variant` sobre él, **Then** no crea ninguna variante adicional — devuelve la
   primera existente sin modificarla (`app/api/v1/catalog/service.py:63-70`).

---

### User Story 4 - SKU automático y su unicidad (Priority: P2)

Un administrador crea una presentación sin especificar SKU manualmente. El sistema genera uno a
partir del nombre del producto y de la presentación, y garantiza que no choque con ningún otro
SKU ya usado en todo el negocio (tenant), no solo dentro del mismo producto.

**Why this priority**: el SKU identifica la presentación en reportes e integraciones; que se
genere solo y sin colisiones evita que el administrador tenga que inventar códigos a mano.

**Independent Test**: se puede probar invocando `_slug` y `_unique_sku` de forma aislada, o
creando dos productos con nombres que generen el mismo prefijo y observando el sufijo que recibe
el segundo.

**Acceptance Scenarios**:

1. **Given** un producto «Cono Waffle» sin SKU explícito, **When** se genera su variante default,
   **Then** `_slug("Cono Waffle")` elimina espacios y caracteres no alfanuméricos, pasa a
   mayúscula y trunca a 4 caracteres → `"CONO"`; el SKU final de la variante «Single» es
   `"CONO-DEF"` (`RN-CAT-06`).
2. **Given** un nombre sin ningún carácter alfanumérico (p. ej. solo símbolos), **When** se
   invoca `_slug`, **Then** el resultado es `"X"` — nunca una cadena vacía (`RN-CAT-06`,
   `app/api/v1/catalog/service.py:16-18`).
3. **Given** que el SKU `"CONO-DEF"` ya existe, **When** se genera el SKU de otra variante con la
   misma base, **Then** `_unique_sku` prueba `"CONO-DEF"` (ocupado) y ofrece `"CONO-DEF-2"`; si
   también está ocupado, `"CONO-DEF-3"`, y así sucesivamente — el sufijo numérico siempre empieza
   en 2, nunca en 1 ni en 0 (`RN-CAT-07`).
4. **Given** que un SKU autogenerado o provisto a mano para la variante de un producto coincide
   con el SKU de una variante de **otro** producto distinto, **When** se intenta guardar,
   **Then** el sistema lo rechaza: la comprobación (`ensure_unique` para SKU manual, o el bucle de
   `_unique_sku` para el autogenerado) consulta `ProductVariant.sku` sin filtrar por
   `product_id` — el SKU es único en **todo el tenant** (`RN-CAT-11`). Un SKU manual que choca
   responde `409 "SKU already exists"` desde `POST /products/{id}/variants` o
   `PATCH /variants/{id}`.

---

### User Story 5 - Nombre de variante duplicado, incluso contra una desactivada (Priority: P1)

Un administrador intenta crear o renombrar una presentación con un nombre que ya usa otra
presentación del mismo producto — sin importar si esa otra sigue activa o fue desactivada
(soft-delete) previamente. El sistema lo rechaza con un `409` que trae los datos necesarios para
que el editor pueda ofrecer «reactivar» en vez de fallar en seco.

**Why this priority**: sin este chequeo, el intento de recrear una presentación desactivada
llegaba hasta el `commit()` y salía como `500` (`UniqueViolation`), un error que el frontend no
puede manejar y que no dice qué hacer — el defecto real que motivó `test_variantes_duplicadas.py`.

**Independent Test**: se puede probar completamente ejecutando
`python -m app.scripts.test_variantes_duplicadas` contra un `pos-backend` en ejecución, sin
depender de ningún otro módulo (usa datos desechables propios).

**Acceptance Scenarios**:

1. **Given** un producto con una variante activa «Pequeña», **When** se intenta crear otra
   variante del mismo producto con nombre `"  pequeña  "` (espacios y mayúsculas distintas),
   **Then** el sistema la recorta y compara en minúsculas, encuentra la coincidencia y responde
   `409` con detalle `{"error": "Ya existe una variante «Pequeña» en este producto",
   "variant_id": "<id de Pequeña>", "active": true}` (`RN-CAT-08`). **Verificado por**
   `test_variantes_duplicadas.py` paso 2 (`_check("señala la variante que estorba", ...)`,
   `_check("y dice que está activa", d["active"], True)`).
2. **Given** esa misma variante «Pequeña» ahora desactivada (`DELETE /variants/{id}`, soft-delete),
   **When** se intenta crear una nueva variante del mismo producto con nombre «Pequeña», **Then**
   el sistema responde `409` con detalle `{"error": "Ya existe una variante «Pequeña»
   desactivada en este producto. Reactívala en vez de crear otra.", "variant_id": "<id de la
   desactivada>", "active": false}` — nunca un `500` (`RN-CAT-08`, `RN-CAT-09`). **Verificado
   por** `test_variantes_duplicadas.py` paso 3 (`_check("con el id de la desactivada", ...)`,
   `_check("y \`active: false\` para ofrecer reactivarla", d["active"], False)`).
3. **Given** dos variantes «Pequeña» y «Mediana» del mismo producto, **When** se intenta
   renombrar «Mediana» a «Pequeña» vía `PATCH /variants/{mediana_id}`, **Then** el sistema
   responde `409` y «Mediana» conserva su nombre original — el mismo chequeo aplica a
   actualización, no solo a creación (`RN-CAT-08`). **Verificado por**
   `test_variantes_duplicadas.py` paso 5.
4. **Given** la variante «Pequeña» desactivada del escenario 2, **When** se ejecuta
   `PATCH /variants/{id} {"active": true}`, **Then** la variante vuelve a estar disponible con su
   receta (`RecipeItem`) intacta — es la única vía de recuperación: reactivar la fila existente,
   no crear una nueva (`RN-CAT-09`). **Verificado por** `test_variantes_duplicadas.py` paso 6.
5. **Given** una carrera entre dos administradores guardando casi al mismo tiempo un mismo
   nombre para el mismo producto, **When** ambos commits llegan a base de datos, **Then** el
   segundo en llegar recibe `409 "Ya existe una variante con ese nombre o SKU"` traducido desde
   el `IntegrityError` de la constraint `uq__product_variants__product_id__name`, no un `500`
   (`app/api/v1/catalog/router.py:70-84`, función `_commit_variante`) — la comprobación previa por
   sí sola no puede cerrar esta carrera.

---

### User Story 6 - Eliminar una variante o una opción nunca borra la fila (Priority: P2)

Un administrador retira una presentación o una opción de la carta. El sistema la desactiva
(`active=False`) en vez de borrarla, preservando el histórico de ventas que la referencian.

**Why this priority**: proteger el histórico de ventas es una obligación de auditoría y
contabilidad; borrar la fila rompería cualquier venta pasada que la referencie por clave foránea.

**Independent Test**: se puede probar llamando `DELETE /variants/{id}` o `DELETE /options/{id}` y
verificando que la fila sigue existiendo en base de datos con `active=False`, en vez de haber
desaparecido.

**Acceptance Scenarios**:

1. **Given** una variante activa, **When** `DELETE /variants/{id}`, **Then** la fila permanece en
   base de datos con `active=False`; no se ejecuta ningún `DELETE` SQL sobre `product_variants`
   (`RN-CAT-10`, `app/api/v1/catalog/router.py:166-181`).
2. **Given** una opción activa, **When** `DELETE /options/{id}`, **Then** la fila permanece con
   `active=False`, y sigue siendo referenciable desde ventas pasadas que la eligieron
   (`RN-CAT-10`, `app/api/v1/catalog/router.py:505-519`).

---

### User Story 7 - Actualizar la imagen de un producto (Priority: P3) — anomalía A-44

Un administrador reemplaza la foto de un producto ya existente. El sistema sube la imagen nueva
(fuera de este servicio), actualiza `image_url` y borra el objeto viejo en Cloudflare R2 — en ese
orden: primero el borrado en R2, después el `db.commit()`.

**Why this priority**: es un flujo de uso frecuente (mantenimiento de catálogo) pero de bajo
riesgo real — el caso que expone la anomalía (fallo del commit justo después del borrado) es
raro, y el borrado en R2 es "best-effort" (nunca lanza excepción).

**Independent Test**: se puede probar llamando `PATCH /products/{id}` con un `image_url` nuevo
sobre un producto que ya tiene imagen, y observando el orden de las llamadas: `delete_object` se
ejecuta antes que `db.commit()`.

**Acceptance Scenarios**:

1. **Given** un producto con `image_url` apuntando a un objeto existente en R2, **When**
   `PATCH /products/{id}` trae un `image_url` distinto al actual, **Then** el sistema asigna el
   `image_url` nuevo al producto en memoria, calcula la `key` del objeto viejo a partir de la URL
   anterior (`key_from_public_url`) y llama `delete_object(old_key)` **antes** de `db.commit()`
   (`products/service.py:78-91`).
2. **Given** el mismo flujo, **When** el `db.commit()` posterior tiene éxito, **Then** el
   comportamiento es correcto de punta a punta: la imagen vieja ya no existe en R2 y
   `product.image_url` en base de datos apunta a la nueva.
3. **Given** el mismo flujo, **When** el `db.commit()` posterior **falla** por cualquier otra
   razón (p. ej. un error de base de datos no relacionado), **Then** `db.rollback()` revierte
   `product.image_url` a la URL vieja en memoria/BD, pero el objeto en R2 que esa URL vieja
   señala **ya fue borrado** un paso antes — el producto queda apuntando a una imagen inexistente,
   sin ningún registro de que ocurrió (`delete_object` es best-effort: solo loguea si falla, nunca
   levanta excepción ni se puede "deshacer"). **Anomalía A-44 (ACCIDENTAL, `registro-riesgos.md`
   R23, severidad Baja)**: el orden de operaciones es verificable por código; el caso de fallo es
   raro. **Tratamiento acordado**: corregir en fase de modernización invirtiendo el orden (commit
   primero, borrado después) o moviendo el borrado a un proceso asíncrono post-commit. Sin riesgo
   retroactivo — documentar el orden actual tal cual, sin corregirlo en esta spec.

---

### Edge Cases

- **Nombre de variante con mayúsculas mixtas y espacios en los extremos**: el schema
  (`str_strip_whitespace=True`) ya recorta antes de validar longitud mínima; la comparación de
  duplicados adicionalmente normaliza a minúsculas y recorta de nuevo por si acaso — «Pequeña »,
  «  pequeña» y «PEQUEÑA» son la misma presentación a efectos de esta regla (`RN-CAT-08`).
- **Datos ya inconsistentes con dos filas que difieren solo en mayúsculas**: `variante_duplicada`
  usa `.first()` sobre el resultado ordenado (activas primero), no `scalar_one_or_none()`,
  precisamente para no reventar con `MultipleResultsFound` si esa situación ya existe en los
  datos (`app/api/v1/catalog/service.py:58-60`).
- **SKU provisto manualmente vs. autogenerado**: un SKU manual se valida con `ensure_unique`
  (409 inmediato si choca); un SKU autogenerado nunca choca porque `_unique_sku` prueba
  sufijos hasta encontrar uno libre — las dos rutas conviven en el mismo endpoint
  (`POST /products/{id}/variants`) según si el body trae `sku` o no.
- **Producto sin ningún carácter alfanumérico en el nombre**: `_slug` no puede fallar ni devolver
  cadena vacía — cae al fallback `"X"` (`RN-CAT-06`).
- **`delete_object` sobre una URL que no pertenece al bucket configurado** (p. ej. una URL vieja
  de otra fuente externa): `key_from_public_url` devuelve `None` y `update_product` no llama a
  `delete_object` en absoluto — solo se borra en R2 cuando la URL vieja es reconocible como
  propia (`app/core/storage.py`).
- **`extra_price` y `price` en cero simultáneamente**: una línea compuesta enteramente de
  elementos gratis calcula un precio de línea de `0` sin error ni caso especial — es la suma
  aritmética normal (`RN-CAT-01`, `RN-CAT-03`, `RN-CAT-04`).

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: El precio de una línea vendida DEBE calcularse como el precio de la variante más la
  suma de `extra_price` de cada opción elegida, sin aplicar descuentos, impuestos ni lógica de
  combos (`RN-CAT-01`).
- **FR-002**: El cálculo del precio de línea NO DEBE aplicar ninguna función de redondeo o
  truncamiento explícito (`round`/`quantize`); el resultado es la suma aritmética exacta de los
  valores `Decimal` de origen, sujeta únicamente a la precisión de columna `Numeric(12,2)` al
  persistir (`RN-CAT-02`).
- **FR-003**: El precio de una variante DEBE ser `>= 0`; un valor negativo DEBE rechazarse con
  `422` antes de tocar base de datos, y adicionalmente está respaldado por un `CHECK` en base de
  datos. Un precio de `0` es válido (`RN-CAT-03`).
- **FR-004**: El `extra_price` de una opción DEBE ser `>= 0`, con el mismo doble respaldo
  (schema + `CHECK` de base de datos), y por defecto es `0` cuando no se especifica (`RN-CAT-04`).
- **FR-005**: Al crear un producto sin ninguna variante, el sistema DEBE crearle automáticamente,
  en la misma operación, una variante vendible `name="Single"`, `price=0`, `active=True` con SKU
  autogenerado. Si el producto ya tiene al menos una variante, esta operación NO DEBE crear una
  adicional (`RN-CAT-05`).
- **FR-006**: Cuando no se provee un SKU explícito, el sistema DEBE generarlo tomando solo
  caracteres alfanuméricos del nombre relevante, convertidos a mayúscula y truncados a **4**
  caracteres; si el nombre no tiene ningún carácter alfanumérico, DEBE usar `"X"` como base
  (`RN-CAT-06`).
- **FR-007**: Si el SKU generado automáticamente ya existe, el sistema DEBE agregar un sufijo
  numérico incremental empezando en **2** (`-2`, `-3`, ...) hasta encontrar uno libre
  (`RN-CAT-07`).
- **FR-008**: El SKU DEBE ser único en todo el tenant (a través de todos los productos), no solo
  dentro de un mismo producto. Un SKU manual que choca con el de otra variante de cualquier
  producto DEBE rechazarse con `409 "SKU already exists"` (`RN-CAT-11`).
- **FR-009**: El sistema DEBE bloquear la creación o el renombrado de una variante a un nombre que
  ya use otra variante del mismo producto, sin importar si esa otra está activa o desactivada. La
  comparación DEBE ser insensible a mayúsculas y a espacios en los extremos. El rechazo DEBE ser
  `409` con un detalle estructurado que incluya `variant_id` y `active` de la variante que ocupa
  el nombre, nunca un `500` de constraint de base de datos (`RN-CAT-08`).
- **FR-010**: Una variante desactivada por soft-delete DEBE seguir "ocupando" su nombre a efectos
  de la validación de FR-009; la única forma de reutilizar ese nombre DEBE ser reactivar la fila
  existente (`PATCH {active: true}`), no crear una nueva. Reactivar DEBE preservar la receta e
  historial asociados a esa variante (`RN-CAT-09`).
- **FR-011**: `DELETE /variants/{id}` y `DELETE /options/{id}` NO DEBEN eliminar la fila de base
  de datos; DEBEN marcar `active=False` exclusivamente, preservando el histórico de ventas que
  las referencian (`RN-CAT-10`).
- **FR-012**: Al actualizar la imagen de un producto (`PATCH /products/{id}` con `image_url`
  distinto al actual), el sistema DEBE borrar el objeto viejo en Cloudflare R2 **antes** de
  ejecutar `db.commit()`. Documentado tal cual — **[Anomalía A-44 — ACCIDENTAL, severidad Baja,
  `registro-riesgos.md` R23]**: si el commit falla después del borrado, `product.image_url`
  revierte a la URL vieja pero el objeto que señala ya no existe en R2, sin registro de que
  ocurrió. **Tratamiento acordado para modernización**: invertir el orden (commit primero,
  borrado después) o mover el borrado a un proceso asíncrono post-commit.

### Key Entities *(include if feature involves data)*

- **Product**: producto del menú (p. ej. «Cono Waffle»). Atributos relevantes a esta spec:
  `name`, `image_url`, `active` (alta/baja del catálogo), `available` (agotado temporal, distinto
  de `active`). El precio no vive aquí, vive en sus variantes.
- **ProductVariant**: presentación vendible de un producto (p. ej. «Pequeña», «Single»).
  Atributos relevantes: `product_id`, `name` (único por producto, insensible a mayúsculas/
  espacios, incluso desactivada), `sku` (único en todo el tenant), `price` (`>= 0`), `active`
  (soft-delete). Es la unidad real de venta — "todo vendible es una variante".
- **Option**: valor dentro de un grupo de opciones (p. ej. un sabor o topping). Atributos
  relevantes a esta spec: `name`, `extra_price` (`>= 0`, default 0), `active` (soft-delete). Su
  vínculo con inventario y su validación de selección pertenecen a las specs 003 y 004
  respectivamente.
- **Objeto en Cloudflare R2**: archivo de imagen del producto, identificado por una `key`
  derivada de `image_url`. Su ciclo de vida (subida, borrado) es externo a la base de datos
  transaccional del sistema, lo que da lugar a la anomalía A-44.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: El 100% de las reglas `RN-CAT-01` a `RN-CAT-11` puede verificarse ejecutando los
  pasos descritos en esta spec contra un `pos-backend` en ejecución, sin necesitar leer
  `catalog/service.py` ni `catalog/router.py` para entender el comportamiento esperado.
- **SC-002**: Un administrador puede dar de alta un producto nuevo y venderlo de inmediato, sin
  ningún paso manual adicional para crear su primera presentación.
- **SC-003**: Un administrador que intenta recrear una presentación desactivada recibe, en la
  misma respuesta `409`, toda la información necesaria (`variant_id`, `active`) para reactivarla
  en un solo paso adicional, sin tener que buscarla manualmente en un listado.
- **SC-004**: `RN-CAT-08` y `RN-CAT-09` quedan cubiertas 1:1 por `test_variantes_duplicadas.py`,
  ejecutable de forma independiente (`python -m app.scripts.test_variantes_duplicadas`) como
  characterization test reproducible.
- **SC-005**: Las nueve reglas restantes de esta spec (`RN-CAT-01` a `RN-CAT-07`, `RN-CAT-10`,
  `RN-CAT-11`) y la anomalía A-44 quedan documentadas con evidencia de código verificable
  (archivo y línea), aun sin tener hoy un characterization test automatizado propio — brecha que
  el equipo de modernización puede usar como lista de pendientes al escribir tests nuevos.

## Out of Scope

- **Qué insumo consume cada línea vendida y cuánto** (receta fija, opciones que descuentan
  inventario, disponibilidad de stock) — cubierto por la spec 003 (`RN-CAT-12` a `RN-CAT-26`,
  `RN-CAT-34`, `RN-CAT-35`).
- **Los grupos de opciones (sabores/toppings) y su validación de selección** (`min_select`/
  `max_select`, `STRICT_OPTION_SELECTION`, unicidad de nombre de opción dentro de su grupo) —
  cubierto por la spec 004 (`RN-CAT-27` a `RN-CAT-33`, `RN-CAT-36` a `RN-CAT-39`).
- **La conversión de unidades de compra** (`app/core/units.py`) — cubierto por la spec 005
  (`RN-CAT-40`, `RN-CAT-41`).

## Assumptions

- **Esta es una spec de ingeniería inversa, no de una feature nueva**: a diferencia del resto de
  las guías de este template ("evitar detalles de implementación"), aquí los endpoints, códigos
  de estado HTTP y nombres de campo **son** el contrato observable que se está documentando — se
  citan explícitamente porque los criterios de aceptación deben ser verificables directamente
  contra `pos-backend` en ejecución o contra `test_variantes_duplicadas.py`.
- **Solo `RN-CAT-08`/`RN-CAT-09` tienen characterization test hoy**: el resto de las reglas de
  esta spec (`RN-CAT-01` a `RN-CAT-07`, `RN-CAT-10`, `RN-CAT-11`) no tiene un script equivalente
  en `app/scripts/`; esta spec documenta su comportamiento a partir de lectura de código y de
  `reglas-de-negocio.md`, no de una ejecución verificada.
- **A-44 es de severidad Baja y no tiene corrección pendiente en esta spec**: se documenta el
  orden actual (borrado en R2 antes del commit) tal cual, sin implementar la corrección acordada
  (invertir el orden o mover a proceso asíncrono) — esa corrección queda para la fase de
  modernización, según Principio I de la Constitución ("el comportamiento actual es sagrado" en
  esta fase de reconocimiento).
- **Los valores numéricos citados (4 caracteres de SKU, sufijo desde 2, etc.) son literales del
  código al momento de esta extracción** (2026-08-16, `pos-backend/app/api/v1/catalog/service.py`).
  Si el código cambia, esta spec debe re-verificarse antes de usarse como characterization test.
