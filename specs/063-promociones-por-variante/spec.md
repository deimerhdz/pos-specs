# Feature Specification: Refactorización del módulo de promociones — modelo por conjunto explícito de variantes

**Feature Branch**: `refactor/063-promociones-por-variante`

**Created**: 2026-08-31

**Status**: Draft

**Naturaleza de esta spec**: **refactorización de comportamiento en producción** (fase de
evolución funcional, Principios I, II, III, VI y VIII de la
[Constitución](../../.specify/memory/constitution.md)). No es una funcionalidad nueva desde
cero: reemplaza el modelo de promociones que hoy corre en producción —caracterizado por las
specs [012](../012-motor-de-evaluacion-de-promociones-y-combos/spec.md) (motor de evaluación) y
[013](../013-administracion-de-promociones-y-combos/spec.md) (administración y máquina de
estados), y ampliado por la spec [040](../040-promociones-precio-por-presentacion/spec.md)
(precio de paquete por presentación, ya mergeada: `pos-backend` PR #47, `pos-heladeria` PR
#48)— por un modelo más simple y expresivo.

Esta spec **lista explícitamente** cada comportamiento actual que cambia (sección "Cambios de
comportamiento respecto de producción", Principio II) y cada test `"CONGELA comportamiento
actual:"` que se ve afectado, con su justificación (Principio III). Las decisiones de negocio
correspondientes se registran en
[`specs/000-reconocimiento/registro-de-anomalias.md`](../000-reconocimiento/registro-de-anomalias.md).
El cambio no es retroactivo: no recalcula ninguna `Sale` / `SaleInvoice` ya emitida (Principio
VII).

**Input**: User description: refactorización del módulo de promociones del POS (`pos-backend`
FastAPI + PostgreSQL schema-per-tenant, `pos-heladeria` Angular). El modelo actual se configura
por producto, pero un producto tiene varias presentaciones con precios distintos; eso impide
expresar los casos reales del cliente (heladería Springfield Granizados). Se pide un modelo que
(1) se configure a nivel de variante, no de producto; (2) soporte descuento por unidad y por
cantidad (paquetes); (3) permita que una misma promoción agrupe varias variantes en un mismo
conjunto elegible, de modo que un paquete pueda combinar variantes distintas que el
administrador eligió explícitamente (p. ej. 1 Ojo de Diablo Pequeño + 1 Perla Negra Pequeño
completan un paquete "2X Pequeños con licor por $12.000"). Se adjuntó el catálogo real del
tenant y sus precios (ver Assumptions).

---

## Estado actual (lo que se reemplaza)

Línea base del refactor, tomada del código en producción:

- **Tipos implementados**: `percent` (% sobre `line_total`), `fixed` (monto fijo por línea),
  `combo` (bundle de selección explícita), `qty_price` (precio de paquete por producto/categoría,
  con precio en cada `PromotionTarget`), `qty_price_presentation` (precio de paquete por
  presentación de catálogo, spec 040). `buy_x_get_y` y `qty_price` figuraron como reservados de
  fase 2; `buy_x_get_y` nunca descontó.
- **Alcance**: `PromotionTarget` apunta a un **producto** o a una **categoría**; una categoría
  marca también los productos creados después. `qty_price_presentation` apunta a una entidad
  **`Presentation`** (catálogo compartido del tenant, creada por la spec 040) vía
  `ProductVariant.presentation_id`.
- **El descuento no se persiste como desglose**: se recalcula al vuelo en cada cobro
  (`combined_discount_detailed()` orquesta motor línea-por-línea + combos + paquete por
  presentación). En la venta solo se guardan el **agregado** `Sale.discount` y un único
  `Sale.promotion_id` (poblado únicamente si todas las líneas descontadas comparten una promo;
  con dos combos o dos promos, queda `NULL` — anomalía A-29).
- **Una sola promoción por línea** (la de menor total tras la reconciliación por prioridad).
- **Prioridad** (`Promotion.priority`, entero, mayor gana; desempate por descuento mayor y luego
  `created_at`) resuelve el conflicto cuando varias promociones tocan la misma línea.
- **Solapamiento**: para `percent`/`fixed`/`qty_price` es **solo advertencia** (`find_overlaps`);
  para `qty_price_presentation` es **bloqueo duro** al crear/activar (spec 040 FR-006).
- **Tipo y alcance/componentes no editables** tras salir de `draft`; solo nombre, valor,
  vigencia y estado. Para cambiar la forma se **duplica**.
- **Estados**: `draft → active → paused → finished`, con transiciones fijas; `finished` es
  terminal.
- **Vigencia**: `starts_at`/`ends_at` (fecha y hora), `days_of_week` (CSV, 0=lunes..6=domingo),
  ventana horaria `start_time`/`end_time` que admite cruzar la medianoche, todo evaluado en la
  zona horaria del tenant. La corrección A-57 (spec 040) atribuye las horas posteriores a la
  medianoche al día en que **inicia** la ventana, para todos los tipos.
- **Tres superficies de consumo**: configuración de administrador, terminal de staff (replica el
  motor en `promotion-pricing.util.ts` solo para preview) y menú QR público (recibe el precio
  con descuento ya resuelto del backend; hoy el menú evalúa asumiendo cantidad 1, así que las
  promociones con mínimo mayor a 1 no se ven hasta el carrito).
- **Entidad `Presentation`** (spec 040): catálogo compartido con módulo de administración propio
  (`/dashboard/presentations`), selector en el formulario de producto, y bloqueo de baja
  mientras una regla activa la referencie.

---

## Clarifications

### Session 2026-08-31

- Q: ¿El modelo de datos debe llevar `type`, `value` y `minQuantity` en cada regla (una regla
  por variante) como propone el diagrama original? → A: No. La regla solo expresa **pertenencia
  al conjunto elegible** (`promotionId`, `productVariantId`). `type`, `value` y `minQuantity`
  viven en la **promoción**, una sola combinación por promoción. Es lo único que permite que un
  paquete combine variantes distintas del conjunto (FR-001, FR-002).
- Q: ¿Un paquete puede mezclar variantes distintas? → A: Sí, siempre que el administrador las
  haya incluido **explícitamente** en el conjunto de esa promoción. El alcance **no es
  automático por presentación**; se deroga la regla "una promo por presentación aplica a todos
  los productos con esa presentación" (FR-003, FR-010).
- Q: ¿Qué significa `fixed_value`? → A: El **precio total del paquete** de `minQuantity`
  unidades. Con `minQuantity = 1` equivale a un precio unitario especial. El descuento es
  (suma de precios normales de las unidades consumidas) − `value` (FR-006).
- Q: `percentaje`/`fixed_value` con `minQuantity > 1`: ¿sobre qué unidades cae el descuento? →
  A: Solo sobre **grupos completos** de `minQuantity` unidades; el remanente se cobra a precio
  normal, para todos los tipos (FR-007).
- Q: Si el conjunto elegible contiene variantes de precio unitario distinto, ¿qué unidades
  consume el paquete? → A: **Consumo codicioso en orden descendente de precio unitario**, con
  desempate por identificador de variante ascendente. Favorece al cliente y es determinista
  (FR-008).
- Q: `combo` en el modelo nuevo → A: Se **elimina** el tipo `combo` y su mecanismo de selección
  explícita. Las promociones `combo` existentes **no se migran automáticamente**: pasan a
  `Finalizada` igual que `qty_price`, y el administrador recrea a mano lo que siga vigente
  (FR-025). Motivo: un `combo` es "esta canasta específica" y el modelo nuevo solo expresa "N
  unidades cualesquiera del conjunto"; migrar en automático cambiaría el precio en silencio.
- Q: Promociones `qty_price` (producto/categoría) y `qty_price_presentation` (presentación)
  vigentes hoy → A: El refactor las lleva a `Finalizada`; el administrador **recrea a mano** lo
  que siga vigente (FR-025).
- Q: Promociones `percent` vigentes hoy → A: **Migración automática**: cada `target`
  de producto o categoría se materializa como un conjunto de variantes (foto fija al momento de
  migrar) (FR-026).
- Q: ¿Cómo se migra una promoción `fixed` (monto fijo de descuento por línea)? → A: **No se
  migra automáticamente**: pasa a `Finalizada` con aviso para recrear a mano, igual que `combo`
  / `qty_price` / `qty_price_presentation`. El `value` de un `fixed` es un monto de descuento,
  no un porcentaje ni un precio de paquete, y copiarlo como precio de paquete cambiaría el
  importe cobrado en silencio (FR-025, FR-026).
- Q: ¿Cómo se redondea y se reparte entre líneas un descuento de **porcentaje** con fracción de
  peso? → A: **Misma regla que precio de paquete** (FR-008a): el descuento del grupo se calcula
  redondeado a peso (FR-006) y se reparte repartiendo el importe cobrado —cada línea cobra
  `floor(precio_normal_del_grupo_en_esa_línea − descuento_grupo × aporte_línea / aporte_total)`—
  y los pesos que falten se suman al importe cobrado de la línea de la variante de identificador
  más alto (desempate por identificador de línea más alto). Garantiza el cuadre al peso de
  SC-005 y es independiente del orden de las líneas.
- Q: En el bloqueo por solapamiento (FR-014), ¿una dimensión no definida (sin franja horaria,
  días vacíos, sin fecha de fin) cuenta como que cubre todo su rango? → A: **Sí**. Sin franja =
  todas las horas; días vacíos = los siete; sin fecha de fin = indefinido. Una dimensión abierta
  se intersecta con cualquier valor de la otra promoción; el bloqueo exige intersección
  simultánea en fecha, días y horas + variante compartida (FR-014a).
- Q: En la terminal del cajero, para una promoción de precio de paquete con `minQuantity` > 1,
  ¿qué ve antes de alcanzar la cantidad mínima? → A: **Igual que el menú QR** (FR-022): la
  condición en lenguaje llano se muestra **siempre** para los productos del conjunto de una
  promoción vigente; el descuento efectivo aparece solo cuando el pedido en curso alcanza
  `minQuantity` unidades elegibles. La terminal nunca aplica el descuento por su cuenta
  (FR-023).
- Q: Tipos reservados `buy_x_get_y` y `qty_price` en el enum del backend → A: Se **retiran**.
- Q: ¿Qué cuenta como "solapamiento no permitido sobre un mismo producto"? → A: Se **bloquea**
  crear o activar una segunda promoción cuyo conjunto comparta al menos una variante con otra
  promoción no terminal **y** cuyas ventanas de fecha **y** día **y** hora **se intersecten**
  (solapamiento real en el tiempo) (FR-014).
- Q: Entonces, ¿para qué queda `priority`? → A: Se **elimina** `priority` del modelo, del
  criterio de conflicto, del listado ordenado por prioridad y de la interfaz. Con el bloqueo de
  solapamiento real de FR-014, dos promociones nunca pueden aplicar a la misma línea en el mismo
  instante, así que no queda ningún conflicto que desempatar.
- Q: `dayOfWeek` (singular en el diagrama) → A: Es un **conjunto** de días de la semana,
  opcional; vacío = todos los días (FR-012).
- Q: Franja horaria (ausente en el diagrama) → A: Se conservan `startTime`/`endTime` opcionales,
  con cruce de medianoche, evaluados en zona horaria del tenant, y se mantiene la corrección
  A-57 (FR-013).
- Q: Edición de una promoción `Activa` → A: Editable solo nombre, descripción, `endAt` y la
  ventana de días/horas; bloqueado el tipo, el valor, la cantidad mínima y el conjunto de
  variantes. Para cambiar eso se duplica (FR-018).
- Q: "Retirar una promoción con permisos" → A: Solo el **administrador del tenant** gestiona
  promociones (crear, editar, duplicar, cambiar de estado). El cajero solo **visualiza** en
  tiempo real qué descuento puede aplicar un producto (FR-019).
- Q: Historial de modificaciones → A: Se **deprecia** — no se construye; si hay algo hoy
  (`audit_logs`, A-42) no se toca ni se expone.
- Q: Persistencia del descuento → A: Se **persiste el monto de descuento agregado** más la lista
  de promociones que lo generaron, en `Sale`, `SaleInvoice` (factura) y `CustomerOrder`, además
  de calcularse al vuelo. Resuelve A-29 (hoy no queda registrada ninguna promoción cuando hay
  más de una). El **desglose por línea de venta** queda fuera de este refactor, para una spec
  aparte (FR-021).
- Q: Menú QR con cantidad 1 y una promoción de `minQuantity = 3` → A: Anuncio permanente con la
  condición en lenguaje llano **y** precio efectivo mostrado cuando el carrito alcanza la
  cantidad (FR-022).
- Q: Analítica de ventas / arqueo → A: Solo **documentar** el efecto; sin requisito nuevo de
  reporte en este refactor.
- Q: Descuento que dejaría el total en cero o negativo → A: El descuento **nunca supera el
  precio normal** de las unidades a las que se aplica; la línea nunca queda por debajo de $0
  (FR-009).
- Q: ¿La entidad `Presentation` sobrevive? → A: No. Se **elimina** el modelo de presentaciones
  (backend y frontend); las reglas de promoción usan directamente `ProductVariant` (FR-027).

---

## User Scenarios & Testing *(mandatory)*

### User Story 1 - El administrador arma una promoción sobre un conjunto explícito de variantes (Priority: P1)

El administrador del tenant crea una promoción, elige un **tipo** (porcentaje o precio de
paquete), un **valor** y una **cantidad mínima**, y luego selecciona **qué variantes** entran
en el conjunto elegible. Para elegirlas dispone de filtros por producto, categoría y texto de
presentación **solo como ayuda de selección**: lo que se guarda es la lista concreta de
variantes, no el filtro. Ve, antes de guardar, un resumen legible: el tipo, la condición
("Llevando 2 pagas $12.000"), y la lista de variantes incluidas con su precio normal.

**Why this priority**: sin esto no existe ninguna promoción que evaluar; es el punto de entrada
de todo lo demás y la razón del refactor (pasar de "por producto" a "por conjunto de
variantes").

**Independent Test**: crear una promoción "2X Pequeños con licor $12.000", agregar al conjunto
las ocho variantes Pequeño con licor del catálogo, ver el resumen, guardar, y comprobar que la
promoción queda en `Borrador` con esas ocho variantes.

**Acceptance Scenarios**:

1. **Given** el administrador en el formulario de nueva promoción, **When** elige tipo "precio
   de paquete", valor $12.000, cantidad mínima 2 y agrega ocho variantes al conjunto, **Then**
   el resumen muestra "Llevando 2 de cualquiera de estas 8 variantes pagas $12.000" y la lista
   de las ocho con su precio normal.
2. **Given** el administrador usa el filtro "categoría = Granizados con licor" para poblar el
   selector, **When** guarda, **Then** se guarda la lista concreta de variantes visibles en ese
   momento; una variante creada después en esa categoría **no** entra sola en la promoción.
3. **Given** una promoción de tipo porcentaje, **When** el administrador intenta guardarla con
   el conjunto de variantes vacío, **Then** el sistema lo impide y explica que una promoción
   necesita al menos una variante.
4. **Given** el administrador define un valor porcentual mayor a 100, **When** intenta guardar,
   **Then** el sistema lo rechaza.
5. **Given** una promoción recién creada, **When** el administrador la revisa, **Then** su
   estado es `Borrador` y todavía no aplica ningún descuento.

---

### User Story 2 - El cajero cobra un pedido y el paquete combina variantes distintas del conjunto (Priority: P1)

Al cobrar, el sistema reúne todas las unidades del pedido cuyas variantes pertenecen al
conjunto de una promoción vigente, arma tantos **grupos completos** de `minQuantity` unidades
como permita el total, y aplica el descuento solo a esos grupos. Las unidades del grupo se
eligen por **consumo codicioso descendente de precio** (favorece al cliente); el remanente se
cobra a precio normal. El resultado no depende del orden de las líneas del pedido.

**Why this priority**: es el problema de negocio central — hoy dos variantes distintas del
mismo tamaño no combinan para un precio de paquete configurado por producto.

**Independent Test**: con "2X Pequeños con licor $12.000" activa (precio normal $8.000 c/u),
cobrar 1 Ojo de Diablo Pequeño + 1 Perla Negra Pequeño y verificar total $12.000; luego 3
unidades y verificar $20.000; en cualquier orden de captura.

**Acceptance Scenarios**:

1. **Given** "2X Pequeños con licor $12.000" activa y un pedido de 1 Ojo de Diablo Pequeño
   ($8.000) + 1 Perla Negra Pequeño ($8.000), **When** se cobra, **Then** el total es $12.000 y
   la terminal muestra la etiqueta de la promoción aplicada.
2. **Given** el mismo pedido con 3 unidades Pequeño con licor de tres variantes distintas,
   **When** se cobra, **Then** el total es $20.000: un grupo de 2 a $12.000 + 1 unidad suelta a
   $8.000; qué unidad queda suelta se decide de forma determinista (variante de identificador
   más alto entre las de igual precio), no por el orden de captura.
3. **Given** el pedido del punto 2 con las líneas capturadas en otro orden, **When** se cobra,
   **Then** el total y el reparto del descuento entre líneas son idénticos.
4. **Given** "10% en granizados" activa (`minQuantity` 1) y un pedido de 1 Granizado Grande con
   licor ($15.000) + 1 Granizado Mediano sin licor ($8.000), ambas variantes en el conjunto,
   **When** se cobra, **Then** el total es $20.700 ($23.000 − $1.500 − $800).
5. **Given** "15% llevando 3 medianos" activa (`minQuantity` 3, conjunto = Mediano con y sin
   licor) y un pedido de 2 Mediano con licor ($11.000) + 2 Mediano sin licor ($8.000), **When**
   se cobra, **Then** el total es $33.500: un grupo de 3 formado por las 3 unidades más caras
   (2×$11.000 + 1×$8.000 = $30.000, −15% = −$4.500), la 4ª unidad ($8.000) a precio normal.
6. **Given** una promoción cuyo pedido no alcanza `minQuantity` en ninguna variante del
   conjunto, **When** se cobra, **Then** no hay descuento ni etiqueta de promoción.
7. **Given** una promoción con `daysOfWeek = {martes}`, **When** el pedido se cobra un
   miércoles, **Then** se cobra sin descuento.
8. **Given** una promoción con ventana horaria 15:00–17:00, **When** el pedido se cobra a las
   14:59, **Then** no hay descuento; **When** se cobra a las 15:01, **Then** sí lo hay.
9. **Given** una variante que quedó desactivada después de agregarse al pedido, **When** se
   cobra, **Then** sus unidades **no** cuentan para formar grupos completos.
10. **Given** "2X Pequeños con licor $12.000" activa y un pedido en curso con **1** unidad de
    una variante del conjunto, **When** el cajero mira ese producto en la terminal, **Then** ve
    la condición "Llevando 2 pagas $12.000" pero todavía sin descuento efectivo; **When** agrega
    la segunda unidad elegible, **Then** la terminal muestra el descuento efectivo de −$4.000
    (FR-023).

---

### User Story 3 - Vigencia por días y franja horaria; sin solapamiento real entre promociones (Priority: P2)

El administrador define en qué días de la semana y en qué franja horaria aplica la promoción
(la franja puede cruzar la medianoche). El sistema impide crear o activar una segunda
promoción que comparta variantes con otra no terminal **si además sus ventanas de fecha, día y
hora se intersectan** — un solapamiento que dejaría a una línea entre dos promociones al mismo
tiempo.

**Why this priority**: sin control de solapamiento real, dos promociones podrían competir por
la misma línea y el resultado sería ambiguo (ya no hay prioridad que lo resuelva).

**Independent Test**: activar "10% en granizados" todos los días; intentar activar "20% en
granizados" también todos los días sobre variantes que se solapan → bloqueado; cambiar la
segunda a `daysOfWeek = {martes}` y ventana 00:00–14:59 mientras la primera va de 15:00 a
cierre → permitido.

**Acceptance Scenarios**:

1. **Given** "Happy hour 22:00–01:00, viernes y sábado, 25% en litros" y un pedido de 1
   Granizado Litro con licor ($28.000), **When** se cobra el sábado a las 00:30 (madrugada),
   **Then** el descuento **sí** aplica: la madrugada del sábado pertenece al viernes, día en
   que inició la ventana (corrección A-57).
2. **Given** una promoción activa cuyo conjunto incluye "Granizado Mediano con licor", **When**
   el administrador intenta activar otra promoción que también incluye esa variante y cuyas
   ventanas de fecha, día y hora se intersectan con la primera, **Then** el sistema lo impide y
   nombra la promoción en conflicto y la variante compartida.
3. **Given** dos promociones que comparten una variante pero con franjas horarias que **no** se
   solapan (una 08:00–15:00, otra 15:00–22:00), **When** el administrador activa la segunda,
   **Then** el sistema lo permite.
4. **Given** una ventana horaria de 22:00 a 02:00, **When** el administrador la guarda, **Then**
   el sistema la acepta como válida (cruza la medianoche).
5. **Given** una promoción activa **sin franja horaria** (todo el día) que incluye "Granizado
   Mediano con licor", **When** el administrador intenta activar otra promoción sobre esa misma
   variante con franja 15:00–17:00 y días/fechas que se intersectan, **Then** el sistema lo
   impide: la promoción sin franja cubre todas las horas, así que se solapa con la de
   15:00–17:00 (FR-014a).

---

### User Story 4 - El cliente del menú QR ve la promoción y su precio efectivo (Priority: P3)

Un cliente que consulta el menú por QR ve, mientras la promoción está vigente en ese momento,
un anuncio con su condición en lenguaje llano ("Llevando 2 pagas $12.000"), aunque no haya
agregado nada al carrito. Cuando el carrito alcanza la cantidad mínima de una variante del
conjunto, el precio efectivo con descuento se refleja en el carrito.

**Why this priority**: valor de descubrimiento; anima a pedir para alcanzar la promoción, pero
no bloquea el cobro correcto (User Story 2).

**Independent Test**: con "2X Pequeños con licor $12.000" activa, abrir el menú QR dentro de su
horario sin agregar nada y ver el anuncio; agregar 2 Pequeños con licor y ver el total $12.000;
volver a abrir fuera de horario y comprobar que el anuncio ya no aparece.

**Acceptance Scenarios**:

1. **Given** una promoción de precio de paquete vigente en ese momento, **When** el cliente abre
   el menú QR sin agregar nada, **Then** ve el anuncio con su condición legible.
2. **Given** una promoción vigente por estado pero fuera de su ventana de día u hora en ese
   momento, **When** el cliente abre el menú QR, **Then** **no** ve el anuncio.
3. **Given** una promoción de `minQuantity` 3 y un carrito con 1 unidad de una variante del
   conjunto, **When** el cliente ve el carrito, **Then** ve la condición ("Llevando 3 pagas
   $X") pero el precio sigue siendo el normal; **When** agrega hasta 3, **Then** el total
   refleja el precio de paquete.

---

### User Story 5 - Duplicar, editar una promoción activa, cambiar de estado (Priority: P2)

El administrador puede duplicar una promoción (la copia nace en `Borrador` con el mismo
conjunto y condición), editar campos no estructurales de una promoción `Activa` (nombre,
descripción, fin de vigencia, días y horas) y moverla por la máquina de estados
`Borrador → Activa → Pausada → Finalizada`. `Finalizada` es terminal.

**Why this priority**: es la operación diaria del administrador; sin duplicar no hay forma de
cambiar el tipo o el conjunto de una promoción que ya estuvo activa.

**Independent Test**: activar una promoción, editar su nombre y su fin de vigencia (permitido),
intentar cambiar su valor y su conjunto de variantes (bloqueado), duplicarla, y en la copia
cambiar el valor.

**Acceptance Scenarios**:

1. **Given** una promoción `Activa`, **When** el administrador edita nombre, descripción, fin
   de vigencia, días u horas, **Then** el cambio se guarda.
2. **Given** una promoción `Activa`, **When** el administrador intenta cambiar el tipo, el
   valor, la cantidad mínima o el conjunto de variantes, **Then** el sistema lo impide y sugiere
   duplicar.
3. **Given** una promoción `Finalizada`, **When** el administrador intenta reactivarla, **Then**
   el sistema lo impide (estado terminal).
4. **Given** una promoción `Activa`, **When** el administrador la duplica, **Then** la copia
   nace en `Borrador` con el mismo conjunto y condición y un nombre distinto.
5. **Given** un usuario cajero, **When** intenta crear o editar una promoción, **Then** el
   sistema lo impide (solo el administrador del tenant gestiona promociones).

---

### User Story 6 - Las promociones existentes en producción se migran o se cierran de forma predecible (Priority: P2)

Al desplegar el refactor, ninguna venta en curso queda sin explicación de su descuento y
ninguna promoción existente desaparece en silencio: las `percent` se migran automáticamente a
conjuntos de variantes (foto fija), y las `fixed` / `combo` / `qty_price` /
`qty_price_presentation` pasan a `Finalizada` con un aviso para que el administrador recree a
mano lo que siga vigente.

**Why this priority**: es una refactorización de algo en producción; una migración ambigua
rompería el cobro o dejaría descuentos fantasma.

**Independent Test**: con una `percent` de categoría, una `fixed`, una `combo` de dos
componentes y una `qty_price_presentation` activas antes del despliegue, correr la migración y
verificar el estado y la forma de cada una después.

**Acceptance Scenarios**:

1. **Given** una promoción `percent` del 10% sobre la categoría "Granizados", **When** se migra,
   **Then** queda como una promoción de tipo porcentaje, valor 10, `minQuantity` 1, con el
   conjunto = todas las variantes activas de esa categoría al momento de migrar, y conserva su
   estado y su vigencia.
2. **Given** una promoción `combo` "1 Granizado Litro + 1 Cono por $30.000", **When** se migra,
   **Then** pasa a `Finalizada` y aparece en el aviso de promociones a recrear; sus líneas de
   venta históricas no cambian.
3. **Given** una promoción `qty_price_presentation` activa, **When** se migra, **Then** pasa a
   `Finalizada` y el administrador ve un aviso de qué promociones debe recrear.
4. **Given** una promoción `fixed` activa ("$2.000 de descuento por línea en granizados"),
   **When** se migra, **Then** pasa a `Finalizada` y aparece en el aviso de promociones a
   recrear; **no** se convierte en una promoción de porcentaje ni de precio de paquete.
5. **Given** una `Sale` ya emitida con un descuento de cualquier tipo, **When** corre la
   migración, **Then** su `discount`, su `total` y su factura **no cambian**.

---

### Edge Cases

- **Remanente mayor que un grupo**: 5 unidades de un conjunto con `minQuantity` 2 → 2 grupos
  completos descontados + 1 unidad suelta a precio normal (FR-007).
- **Conjunto con precios unitarios distintos**: si el conjunto incluye variantes de $8.000 y de
  $6.000, el grupo consume primero las de $8.000 (codicioso descendente); qué unidad concreta,
  con precios iguales, se decide por identificador de variante ascendente (FR-008). Ver ejemplo
  numérico en Assumptions.
- **Precio de paquete que no representa descuento**: si el peor caso del conjunto
  (`minQuantity` unidades de la variante **más barata** del conjunto) cuesta menos o igual que
  el precio de paquete configurado, guardar la promoción se **bloquea** — no basta con advertir
  (FR-016). En el cobro, un grupo nunca se aplica si dejaría la línea con un total mayor que sin
  promoción (FR-009).
- **Promoción que expira o se pausa con sesión de mesa abierta / carrito lleno**: el siguiente
  cálculo de cobro simplemente ya no la encuentra vigente y no aplica su descuento; no existe un
  "descuento congelado" de una promoción pausada (FR-020).
- **Menú QR con cantidad 1 y `minQuantity` 3**: se muestra siempre la condición legible; el
  precio efectivo solo cambia al alcanzar la cantidad en el carrito (FR-022).
- **Descuento que dejaría el total en cero o negativo**: el descuento se topa en el precio
  normal de las unidades a las que aplica; la línea nunca baja de $0 (FR-009).
- **Variante eliminada del catálogo**: una variante borrada sale del conjunto de toda promoción
  que la incluyera; si el conjunto queda vacío, la promoción deja de aplicar descuento pero no
  cambia de estado sola (FR-011).
- **Edición de una promoción activa**: quedan bloqueados tipo, valor, cantidad mínima y conjunto
  de variantes, porque una promoción que ya explicó el descuento de una venta reescribiría esa
  historia si cambiara de forma (FR-018).
- **Ventana horaria que cruza la medianoche con días restringidos**: las horas posteriores a la
  medianoche pertenecen al día de inicio de la ventana para evaluar `daysOfWeek` y `endAt`
  (corrección A-57, se mantiene) (FR-013).
- **Solapamiento parcial en el tiempo**: dos promociones que comparten una variante pero cuyas
  ventanas horarias no se intersectan pueden coexistir activas (FR-014).
- **Solapamiento con una dimensión abierta**: una promoción sin franja horaria (o sin días
  restringidos, o sin fecha de fin) cubre todo ese dominio, así que se intersecta con cualquier
  franja/día/fecha de otra promoción sobre una variante compartida y el bloqueo aplica
  (FR-014a).

---

## Requirements *(mandatory)*

### Modelo y configuración

- **FR-001**: Una promoción DEBE tener exactamente **una** combinación de (tipo, valor, cantidad
  mínima) y un **conjunto** de una o más variantes de producto. La pertenencia de una variante
  al conjunto NO lleva ningún parámetro de precio propio.
- **FR-002**: Los tipos DEBEN ser **porcentaje** (`value` = porcentaje de descuento, 0 <
  `value` ≤ 100) y **precio de paquete** (`value` = precio total en COP de `minQuantity`
  unidades). No DEBE existir ningún otro tipo. Los identificadores `buy_x_get_y`, `qty_price`,
  `qty_price_presentation`, `combo` y `fixed` se retiran del sistema (`buy_x_get_y` y
  `qty_price` estaban reservados; `qty_price_presentation`, `combo` y `fixed` estaban
  implementados — ver FR-024, FR-025).
- **FR-003**: El alcance de una promoción DEBE resolverse **solo** por la lista explícita de
  variantes del conjunto. NO DEBE existir alcance por producto, por categoría, ni por
  presentación, ni ningún mecanismo que incluya automáticamente variantes creadas después.
- **FR-004**: La interfaz de administración DEBE ofrecer filtros por producto, categoría y texto
  de presentación **únicamente como ayuda para poblar el selector de variantes**; lo que se
  guarda es la lista concreta de variantes.
- **FR-005**: Antes de confirmar la creación o la edición, la interfaz DEBE mostrar un resumen
  legible: el tipo, la condición en lenguaje llano y la lista de variantes del conjunto con su
  precio normal vigente.
- **FR-006**: Para el tipo **precio de paquete**, el descuento de un grupo completo DEBE ser
  (suma de los precios normales de las `minQuantity` unidades que el grupo consume) − `value`.
  Con `minQuantity` = 1, la promoción DEBE comportarse como un precio unitario especial de
  `value`. Para el tipo **porcentaje**, el descuento de un grupo completo DEBE ser
  `value` % × (suma de los precios normales de las `minQuantity` unidades que el grupo consume),
  redondeado a peso.
- **FR-007**: Para **ambos** tipos, el descuento DEBE aplicarse ÚNICAMENTE a grupos completos de
  `minQuantity` unidades elegibles (`total_unidades_elegibles // minQuantity` grupos). Toda
  unidad sobrante DEBE cobrarse a precio normal.
- **FR-008**: Cuando el conjunto contiene variantes de precio unitario distinto, las unidades
  que forman los grupos completos DEBEN elegirse por **precio unitario descendente** (el grupo
  consume primero las más caras), con desempate por identificador de variante ascendente y luego
  por identificador de línea ascendente, de modo que el total y el reparto por línea NO dependan
  del orden de las líneas del pedido.
- **FR-008a**: El descuento de un grupo (FR-006, redondeado a peso) DEBE repartirse entre sus
  líneas contribuyentes, para **ambos** tipos, repartiendo el **importe cobrado**: cada línea
  contribuyente cobra `floor(precio_normal_de_sus_unidades_del_grupo − descuento_grupo ×
  aporte_de_la_línea / aporte_total_del_grupo)`; los pesos que falten para llegar al importe que
  el grupo debe cobrar (`aporte_total − descuento_grupo`) se suman al importe cobrado de la
  línea de la variante de **identificador más alto** (desempate: identificador de línea más
  alto). Esa línea absorbe así el ajuste de redondeo, coherente con FR-008 (esa variante es la
  última en entrar al grupo). La suma de los descuentos por línea DEBE cuadrar exactamente, al
  peso, con el descuento del grupo (SC-005), sin importar el orden de las líneas.
- **FR-009**: El descuento aplicado a una línea NUNCA DEBE superar el precio normal de las
  unidades de esa línea a las que se aplica; ninguna línea DEBE quedar con un total menor que
  $0. En el cobro, un grupo NO DEBE aplicarse a una línea si el resultado deja a esa línea con
  un total **mayor** que sin promoción.
- **FR-010**: Un paquete DEBE poder formarse con cualquier combinación de unidades cuyas
  variantes pertenezcan al conjunto de la promoción, sin importar de qué producto provenga cada
  unidad. Se deroga la regla "una promoción por presentación aplica a todos los productos con
  esa presentación".
- **FR-011**: Una variante **desactivada** o **eliminada** NO DEBE contar como unidad elegible.
  Si el conjunto de una promoción activa queda sin ninguna variante elegible, la promoción DEJA
  de aplicar descuento pero NO cambia de estado automáticamente.

### Vigencia

- **FR-012**: La promoción DEBE tener: nombre, descripción opcional, fecha de inicio
  (obligatoria), fecha de fin (opcional), un **conjunto** de días de la semana en los que
  aplica (opcional; vacío = todos los días) y hora de inicio y fin (opcionales, ambas o
  ninguna).
- **FR-013**: Cuando la promoción define hora de inicio y fin, el sistema DEBE permitir que la
  ventana cruce la medianoche (p. ej. 22:00–02:00). Días y horas se evalúan en la zona horaria
  del tenant. Cuando la ventana cruza la medianoche, las horas posteriores a la medianoche DEBEN
  atribuirse al día en que **inicia** la ventana para evaluar `daysOfWeek` y la fecha de fin
  (corrección A-57, se conserva).
- **FR-014**: El sistema NO DEBE permitir crear ni activar una promoción cuyo conjunto comparta
  al menos una variante con otra promoción en estado `Borrador`, `Activa` o `Pausada` **si
  además** sus rangos de fecha, sus conjuntos de días y sus ventanas horarias **se
  intersectan**. El mensaje DEBE nombrar la promoción en conflicto y la(s) variante(s)
  compartida(s). Si las ventanas NO se intersectan, la coexistencia DEBE permitirse.
- **FR-014a**: Al evaluar la intersección de FR-014, una dimensión **no definida** DEBE tratarse
  como que cubre todo su dominio: sin franja horaria = 00:00–24:00 (todas las horas); conjunto
  de días vacío = los siete días; sin fecha de fin = vigencia indefinida. Una dimensión abierta
  se intersecta con cualquier valor de esa misma dimensión en la otra promoción. El bloqueo solo
  se produce cuando las **tres** dimensiones (fecha, días y horas) se intersectan a la vez y hay
  al menos una variante compartida.

### Estados, edición, permisos

- **FR-015**: Los estados DEBEN ser `Borrador`, `Activa`, `Pausada`, `Finalizada`, con
  transiciones `Borrador → {Activa, Finalizada}`, `Activa → {Pausada, Finalizada}`,
  `Pausada → {Activa, Finalizada}`, `Finalizada → {}` (terminal). Solo `Activa` habilita el
  descuento.
- **FR-016**: Al guardar (crear o duplicar-y-editar antes de activar), si el tipo es **precio de
  paquete** y `value` ≥ (`minQuantity` × precio normal de la variante **más barata** del
  conjunto), el sistema DEBE **bloquear** el guardado y explicar que la promoción no
  representaría un descuento para al menos una combinación del conjunto.
- **FR-017**: El sistema DEBE permitir **duplicar** una promoción. La copia nace en `Borrador`
  con el mismo tipo, valor, cantidad mínima, conjunto de variantes y vigencia, y un nombre
  distinto.
- **FR-018**: En estado `Activa` o `Pausada`, DEBE ser editable únicamente: nombre, descripción,
  fecha de fin, conjunto de días y ventana horaria. DEBEN estar bloqueados: tipo, valor,
  cantidad mínima y conjunto de variantes. En `Borrador` todo es editable.
- **FR-019**: Solo el **administrador del tenant** DEBE poder crear, editar, duplicar, cambiar
  de estado o eliminar una promoción. El cajero DEBE poder **visualizar** en la terminal, en
  tiempo real, qué descuento puede aplicar cada producto, sin poder modificarlo.
- **FR-020**: El descuento NUNCA DEBE persistirse como un valor congelado que sobreviva a un
  cambio de estado de la promoción: cada cálculo de cobro DEBE partir del estado actual del
  pedido y de las promociones vigentes en ese momento. (La persistencia de FR-021 es un
  **registro del resultado ya cobrado**, no una congelación previa.)

### Persistencia del resultado y superficies de consumo

- **FR-021**: Al emitir una venta, el sistema DEBE registrar el **monto de descuento agregado**
  efectivamente aplicado, más la **lista de promociones** que lo generaron, en `Sale`, en la
  factura (`SaleInvoice`) y en `CustomerOrder`, de forma que el arqueo de caja y la consulta de
  una venta pasada muestren el mismo descuento que se cobró y de qué promoción(es) vino. Esto
  resuelve la anomalía A-29 (hoy, con más de una promoción o combo, no queda registrada
  ninguna). El **desglose por línea de venta** (variante ↔ promoción ↔ monto) queda **fuera de
  alcance** de este refactor. El registro NO es retroactivo: no altera ninguna venta ni factura
  emitida antes del despliegue.
- **FR-022**: El menú QR público DEBE mostrar, para cada promoción vigente en ese momento según
  su ventana de día y hora (zona horaria del tenant), un anuncio con su condición legible (p.
  ej. "Llevando 2 de estos sabores pagas $12.000"), sin que el cliente agregue nada al carrito.
  Cuando el carrito alcanza `minQuantity` unidades elegibles, el precio efectivo con descuento
  DEBE reflejarse en el carrito. Fuera de la ventana, el anuncio NO se muestra.
- **FR-023**: La terminal de staff DEBE mostrar, para un producto/variante del conjunto de una
  promoción vigente, **siempre** su condición en lenguaje llano ("Llevando 2 pagas $12.000"),
  aunque el pedido en curso todavía no alcance `minQuantity` unidades elegibles. Cuando el
  pedido en curso **sí** alcanza `minQuantity` unidades elegibles, la terminal DEBE mostrar
  además el **descuento efectivo**, calculado con el mismo criterio que usa el cobro (mismo
  comportamiento que el menú QR, FR-022). La terminal NO DEBE aplicar el descuento por su
  cuenta: el cálculo vinculante es el del cobro.

### Migración de lo existente

- **FR-024**: El tipo `combo` y su mecanismo de **selección explícita** (marca de combo en las
  líneas de carrito, de pedido y de venta; expansión de un combo en sus componentes al
  agregarlo) DEBEN retirarse por completo de la terminal, del carrito y del cobro. Las líneas de
  venta históricas que quedaron marcadas con un combo NO DEBEN alterarse.
- **FR-025**: Cada promoción existente de tipo `combo`, `qty_price` (producto/categoría),
  `qty_price_presentation` (presentación) o `fixed` (monto fijo de descuento por línea) DEBE
  pasar a `Finalizada` durante la migración. NO se migra automáticamente ninguna a la forma
  nueva: un `combo` es una canasta específica de componentes, un `qty_price` lleva precio por
  producto o por presentación, y el `value` de un `fixed` es un monto de descuento por línea que
  no equivale ni a un porcentaje ni a un precio de paquete; traducir cualquiera de ellos en
  automático a "N unidades cualesquiera del conjunto" cambiaría el precio en silencio. El
  administrador DEBE ver un aviso con la lista de promociones que quedaron finalizadas para
  recrearlas si siguen vigentes.
- **FR-026**: Cada promoción `percent` existente DEBE migrarse conservando su tipo (porcentaje),
  su valor, su vigencia y su estado, con el conjunto = todas las variantes activas alcanzadas
  por sus `targets` al momento de migrar (foto fija). Una `percent` global (sin `targets`) DEBE
  migrarse con el conjunto = todas las variantes activas del tenant al momento de migrar. Las
  promociones `fixed` NO se migran automáticamente: pasan a `Finalizada` (FR-025).
- **FR-027**: El modelo de **presentaciones** (entidad de catálogo, su módulo de administración,
  el selector en el formulario de producto y la referencia desde la variante) DEBE eliminarse
  por completo en backend y frontend. Ninguna funcionalidad DEBE depender de él tras el refactor.
- **FR-028**: Ningún test con el prefijo `"CONGELA comportamiento actual:"` DEBE modificarse sin
  que esta spec lo nombre y justifique (ver "Cambios de comportamiento respecto de producción").

### Key Entities

- **Promoción**: nombre, descripción opcional, tipo (porcentaje | precio de paquete), valor,
  cantidad mínima, fecha de inicio, fecha de fin opcional, conjunto de días de la semana
  opcional, ventana horaria opcional (con cruce de medianoche), estado
  (`Borrador`/`Activa`/`Pausada`/`Finalizada`). Ya **no** tiene prioridad, ni alcance por
  producto/categoría, ni componentes de combo, ni reglas de presentación.
- **Pertenencia al conjunto**: relación entre una promoción y una variante de producto. No lleva
  ningún atributo de precio. Una promoción tiene N; una variante puede pertenecer a varias
  promociones siempre que sus ventanas no se intersecten (FR-014).
- **Variante de producto**: unidad vendible con su precio. Su estado activo/inactivo determina
  si cuenta para formar grupos completos. Ya **no** referencia ninguna presentación de catálogo.
- **Grupo completo (paquete)**: agrupación de `minQuantity` unidades elegibles del conjunto,
  formada al momento de cobrar por consumo codicioso descendente de precio. Solo los grupos
  completos generan descuento.
- **Registro de descuento en la venta**: el monto de descuento efectivamente aplicado, guardado
  en `Sale`, `SaleInvoice` y `CustomerOrder` al emitir (FR-021).

---

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: El 100% de los pedidos que reúnen `minQuantity` unidades de variantes del conjunto
  de una promoción vigente reciben el descuento correspondiente, sin importar de qué producto
  sean las unidades ni el orden de las líneas.
- **SC-002**: El 100% de los intentos de guardar una promoción de precio de paquete cuyo peor
  caso no representa un descuento son rechazados con una explicación clara.
- **SC-003**: El 100% de los intentos de crear o activar una promoción que se solaparía en el
  tiempo con otra sobre una variante compartida son rechazados nombrando el conflicto; el 100%
  de los que no se solapan en el tiempo se permiten.
- **SC-004**: El 0% de los pedidos cobrados fuera del día o del horario configurado de una
  promoción reciben su descuento.
- **SC-005**: En el 100% de los cobros con grupos completos, la suma de los descuentos por línea
  cuadra exactamente, al peso, con el descuento total, sin importar el orden de las líneas.
- **SC-006**: El 100% de las promociones existentes en producción al desplegar el refactor
  quedan, tras la migración, en un estado explícito (migrada y activa/pausada, o finalizada con
  aviso), y ninguna `Sale` ni factura previa cambia de importe.
- **SC-007**: Un cliente del menú QR puede identificar la condición de una promoción vigente sin
  agregar nada al carrito, y no ve anunciadas las promociones fuera de su ventana de día u hora.
- **SC-008**: Un administrador puede crear la promoción "2X Pequeños con licor $12.000" sobre
  las ocho variantes Pequeño con licor y verificar, cobrando 1 Ojo de Diablo Pequeño + 1 Perla
  Negra Pequeño, un total de exactamente $12.000, sin leer código.

---

## Cambios de comportamiento respecto de producción (Principio II y III)

Esta refactorización cambia comportamiento observable de promociones existentes. Cada punto
requiere una entrada en
[`registro-de-anomalias.md`](../000-reconocimiento/registro-de-anomalias.md) (decisión de
negocio, fase de plan):

1. **Se elimina la prioridad** (`Promotion.priority`) por completo: del modelo, del criterio de
   resolución de conflicto (RN-PROMO-13, RN-PROMO-14), del listado ordenado por prioridad
   (RN-PROMO-43) y de la interfaz. En su lugar, el solapamiento real en el tiempo sobre una
   variante compartida se **bloquea** al crear/activar (FR-014), así que no queda ningún
   conflicto que desempatar.
2. **El solapamiento pasa de advertencia a bloqueo** para todas las promociones automáticas
   (RN-PROMO-30), con el criterio acotado de FR-014 (variante compartida + ventanas que se
   intersectan).
3. **Se elimina el alcance por categoría** y la regla "marcar una categoría incluye los
   productos creados después" (RN-PROMO-06 en su parte de categoría; regla de negocio de la
   sección 4 de la solicitud original, derogada).
4. **Se elimina el tipo `combo`** y su mecanismo de selección explícita (`combo_id` en líneas de
   carrito, orden y venta; expansión al agregarlo) (FR-024). Las promociones `combo` vigentes
   pasan a `Finalizada` (FR-025); **no** se migran a otra forma, para no cambiar un precio en
   silencio. Las líneas de venta históricas marcadas con un combo no se tocan.
5. **Se eliminan los tipos `qty_price` por producto/categoría, `qty_price_presentation` por
   presentación y `fixed` (monto fijo de descuento por línea)**; las instancias vigentes pasan a
   `Finalizada` (FR-025). El `value` de un `fixed` es un descuento, no un porcentaje ni un
   precio de paquete, así que **no** se migra a otra forma para no cambiar un precio en
   silencio. Solo `percent` se migra automáticamente (FR-026).
6. **Se elimina la entidad `Presentation`** y todo lo que depende de ella (spec 040 revertida en
   su parte de modelo de datos; el resto de spec 040 —vigencia por día/hora, cruce de
   medianoche, anuncio en menú QR— se conserva) (FR-027).
7. **Se persiste el descuento agregado + la lista de promociones que lo generaron** en `Sale`,
   `SaleInvoice` y `CustomerOrder` (hoy solo el agregado en `Sale.discount` y un único
   `promotion_id` que queda `NULL` con más de una promoción; `CustomerOrder` no tiene campo de
   descuento) (FR-021). Resuelve A-29. El desglose por línea de venta queda fuera de alcance.
8. **La regla "dentro de una misma presentación el precio unitario es siempre el mismo" se
   deroga** — es falsa en el catálogo real (un Pequeño con licor cuesta $8.000 y uno sin licor
   $6.000). No DEBE usarse como invariante en ninguna validación (ver Assumptions).

### Tests `"CONGELA comportamiento actual:"` afectados

Todos requieren reescritura; la justificación es el cambio de modelo aprobado en esta spec. El
**comportamiento congelado que sigue vigente** (una sola promoción por línea, descuento tope al
precio normal, remanente a precio normal, vigencia en hora local, cruce de medianoche) se
**re-congela** con casos equivalentes en el nuevo modelo.

| Test | Archivo | Motivo del cambio |
|---|---|---|
| `test_add_item_combo` | `test_cart_service.py` | Se elimina la selección de combo (FR-024). |
| `test_serialize_cart_combo_no_recibe_descuento_adicional` | `test_cart_service.py` | Ídem. |
| `test_serialize_cart_discounted_total_con_promocion_activa` | `test_cart_service.py` | Usa alcance por **categoría**; se reescribe con conjunto de variantes (FR-003). |
| `test_serialize_cart_discounted_total_sin_promocion` | `test_cart_service.py` | Se conserva el invariante (`discounted_total` = `None` sin promo); se revisa el montaje. |
| `test_add_item_to_table_combo_expande_componentes_a_precio_normal` | `test_orders_consolidation.py` | Se elimina la expansión de combo (FR-024). |
| `test_close_session_unified_a29_promotion_id_no_registra_combos_multiples` | `test_table_sessions_service.py` | A-29: cambia con la persistencia del agregado + lista de promociones (FR-021) y sin combos. |
| `test_promo_lines_for_camino_feliz_y_sin_promocion_aplicable` | `test_orders_checkout.py` | El motor cambia de `evaluate` a un cálculo por conjunto; se re-congela el resultado. |
| `test_pay_order_construye_sale_real_con_promocion_activa` | `test_orders_checkout.py` | Ídem; el `Sale` ahora registra el descuento agregado + promociones (FR-021). |
| `test_pay_order_dos_combos_distintos_a29_promotion_id_none` | `test_orders_checkout.py` | Se elimina combo; A-29 se resuelve por otra vía (FR-021). |
| `test_submit_cart_snapshot_de_descuento_coincide_con_el_carrito` | `test_cart_service.py` | Spec 038 (no CONGELA): el snapshot de descuento en `OrderItem` debe seguir cuadrando con el nuevo motor. |

Los tests de la spec 040 (`test_promotions_presentation_pricing.py`,
`test_promotions_presentation_rules.py`, `test_presentations_service.py`) **no llevan prefijo
CONGELA** y se **eliminan** junto con la entidad de presentación.

Todo test `"CONGELA comportamiento actual:"` que congele el descuento del tipo `fixed` (monto
fijo por línea) o del tipo `qty_price` también queda afectado: esas promociones pasan a
`Finalizada` en la migración (FR-025) y dejan de descontar. El inventario concreto de esos
tests, con su nombre y archivo, se completa en `/speckit-plan`.

---

## Out of Scope

- El tipo "compra X y lleva Y" (`buy_x_get_y`).
- Cupones o códigos promocionales.
- Promociones por cliente o por segmento.
- Acumulación de varias promociones sobre una misma línea (sigue habiendo **una** por línea).
- Descuento manual del cajero (ya rechazado por spec 029; sin cambios).
- Un reporte nuevo de analítica de ventas por promoción (solo se documenta el efecto; ver G3).
- Recalcular retroactivamente ventas o facturas ya emitidas.
- El detalle de columnas, tablas y migraciones concretas (corresponde a `/speckit-plan`,
  Principio VIII).

---

## Assumptions

- **Catálogo real del tenant (Springfield Granizados).** Presentaciones: Pequeños, Medianos,
  Grandes, Extra grandes, Baldes, Litros. Cada presentación existe en variante **con licor** y
  **sin licor**, a precios distintos:

  | Presentación | Con licor | Sin licor | Precio "2X" (lun–jue, solo con licor) |
  |---|---|---|---|
  | Pequeños | $8.000 | $6.000 | $12.000 |
  | Medianos | $11.000 | $8.000 | $17.000 |
  | Grandes | $15.000 | $11.000 | $22.000 |
  | Extra grandes | $18.000 | $13.000 | $27.000 |
  | Baldes | $21.000 | $15.000 | $31.000 |
  | Litros | $28.000 | $19.000 | $41.000 |

  Los ocho sabores Pequeño con licor citados en la solicitud: Ojo de Diablo, Red Fantasy, Fresa
  Boom, Mora Azul, Martini Manzana, Margarita Tahiti, Perla Negra, Caipiriña.

- **La regla "precio uniforme por presentación" es falsa** y se deroga (ver punto 8 de "Cambios
  de comportamiento"). Ninguna validación debe asumir que dos variantes del mismo tamaño cuestan
  lo mismo.

- **Los seis "2X" son seis promociones separadas**, una por presentación, todas con
  `daysOfWeek = {lunes, martes, miércoles, jueves}`, tipo precio de paquete, `minQuantity` 2, y
  conjunto = las variantes **con licor** de esa presentación. No se solapan entre sí porque sus
  conjuntos son disjuntos. Si el conjunto incluyera variantes **sin licor**, FR-016 bloquearía
  el guardado en Medianos, Extra grandes, Baldes y Litros (2 sin licor cuestan menos que el
  precio 2X).

### Ejemplos numéricos que cuadran exacto (COP)

- **Porcentaje, `minQuantity` 1** — "10% en granizados", conjunto = variantes de granizado.
  Pedido: 1 Grande con licor ($15.000) + 1 Mediano sin licor ($8.000). Cada unidad es su propio
  grupo. Descuento línea 1 = $1.500; línea 2 = $800. Total: $23.000 − $2.300 = **$20.700**.

- **Porcentaje, `minQuantity` 3, solo grupos completos** — "15% llevando 3 medianos", conjunto =
  {Mediano con licor, Mediano sin licor}. Pedido: 2 Mediano con licor ($11.000) + 2 Mediano sin
  licor ($8.000) = 4 unidades, $38.000. Grupos: 4 // 3 = 1. Consumo codicioso: el grupo toma las
  3 más caras (2×$11.000 + 1×$8.000 = $30.000), descuento 15% = $4.500. 4ª unidad ($8.000) a
  precio normal. Total: $38.000 − $4.500 = **$33.500**. Reparto: línea "con licor" (2 u en el
  grupo) −$3.300; línea "sin licor" (1 u en el grupo, 1 suelta) −$1.200. Suma −$4.500. ✔

- **Precio de paquete, `minQuantity` 1** — "Litro sin licor a $17.000 los lunes" (normal
  $19.000), conjunto = {Litro sin licor}. Pedido: 2 unidades. 2 // 1 = 2 grupos, cada uno a
  $17.000. Total: **$34.000**; descuento $38.000 − $34.000 = $4.000. ✔

- **Precio de paquete, `minQuantity` 2, conjunto mixto** — "2X Pequeños con licor $12.000",
  conjunto = las 8 variantes Pequeño con licor ($8.000 c/u). Pedido: 1 Ojo de Diablo Pequeño + 1
  Perla Negra Pequeño = 2 u, $16.000 normal. 2 // 2 = 1 grupo a $12.000. Total: **$12.000**;
  descuento $4.000. Reparto: $12.000 ÷ 2 = $6.000 por unidad, cada línea (1 u) se cobra $6.000
  (−$2.000). Suma −$4.000. ✔

- **Remanente** — mismo caso, pedido de 3 u (2 Ojo de Diablo + 1 Fresa Boom), $24.000 normal.
  3 // 2 = 1 grupo (2 u a $6.000) + 1 suelta ($8.000). Total: $12.000 + $8.000 = **$20.000**;
  descuento $4.000. La unidad suelta sale de la variante de identificador más alto entre las de
  igual precio (determinista).

- **Redondeo de división no exacta** (regla ilustrativa, no un "2X" real; regla de
  redondeo/residuo de FR-008a, válida para **ambos** tipos) — "3 Pequeños sin
  licor por $16.000", conjunto = variantes Pequeño sin licor ($6.000 c/u). Pedido: 3 u de
  sabores distintos = $18.000 normal. 1 grupo. Importe cobrado por unidad = `floor($6.000 −
  $2.000 × 6.000/18.000)` = `floor($5.333,33)` = $5.333; falta $16.000 − 3×$5.333 = $1 para el
  importe del grupo, que se suma al importe cobrado de la unidad de identificador de variante
  más alto → esa unidad se cobra $5.334, las otras dos $5.333. Suma $16.000. Descuento total:
  $18.000 − $16.000 = **$2.000**. ✔

- **Reparto de un descuento de porcentaje entre líneas** (FR-008a) — "15% llevando 3 medianos",
  grupo = 2 Mediano con licor ($11.000) + 1 Mediano sin licor ($8.000), base $30.000, descuento
  15% = $4.500. Reparto proporcional: línea con licor `floor($22.000 − $4.500 × 22.000/30.000)`
  = $18.700 (descuento $3.300); línea sin licor `floor($8.000 − $4.500 × 8.000/30.000)` = $6.800
  (descuento $1.200). Suma de descuentos $4.500, sin residuo. ✔ (es el escenario 5 de la User
  Story 2).

- **Conjunto mal armado con precios distintos** (caso límite nuevo) — conjunto = Pequeños con y
  sin licor (bloqueado por FR-016 al guardar, pero si estuviera activo). Pedido: 1 Ojo de Diablo
  Pequeño ($8.000) + 1 Manzana Verde Pequeño sin licor ($6.000) + 1 Perla Negra Pequeño
  ($8.000), `minQuantity` 2, precio de paquete $12.000. Consumo codicioso descendente: el grupo
  toma las dos de $8.000 → paquete $12.000; la de $6.000 queda suelta. Total: $12.000 + $6.000 =
  **$18.000** (no $20.000). Documentado como regla de negocio con este ejemplo.

- **Ventana que cruza la medianoche** — "Happy hour 22:00–01:00, viernes y sábado, 25% en
  litros". Pedido: 1 Granizado Litro con licor ($28.000), cobrado el **sábado 00:30**. La
  madrugada del sábado pertenece al **viernes** (día de inicio de la ventana, A-57), y el
  viernes está en `daysOfWeek`, así que aplica: descuento $7.000, total **$21.000**.

### Otras asunciones

- **Persistencia (FR-021)**: el registro en `Sale` / `SaleInvoice` / `CustomerOrder` es el
  **monto de descuento agregado** más la lista de promociones que lo generaron. El desglose por
  línea de venta (variante ↔ promoción ↔ monto) NO se construye en este refactor; si se
  necesita para auditoría o analítica, es una spec aparte. La forma concreta de guardar "lista
  de promociones" (columna nueva, tabla puente) es decisión de `/speckit-plan`.
- **Menú QR**: se reutiliza el mecanismo de anuncio introducido por la spec 040 (FR-021 de esa
  spec), adaptado a "conjunto de variantes" en vez de "presentación".
- **La terminal de staff** sigue teniendo una réplica del criterio de elegibilidad y de cálculo
  solo para preview; el cálculo vinculante es el del cobro (FR-023), como hoy.
- **Zona horaria**: por tenant (A-46 / spec 030), no un valor global de instancia.
- **Rama de trabajo**: `refactor/063-promociones-por-variante` (063 es el siguiente número
  libre; el `061` del ejemplo de la solicitud ya está tomado por
  `061-notas-visibles-mis-pedidos`).

### Decisiones tomadas en la sesión de clarificación (2026-08-31)

- **Prioridad**: se elimina por completo (no queda como desempate).
- **Combos existentes**: pasan a `Finalizada` y se recrean a mano; sin migración automática.
- **Promociones `fixed` existentes**: pasan a `Finalizada` y se recrean a mano; sin migración
  automática (solo `percent` se migra). Su `value` es un monto de descuento, no un precio de
  paquete ni un porcentaje.
- **Persistencia**: monto agregado + lista de promociones, en `Sale` / `SaleInvoice` /
  `CustomerOrder`; el desglose por línea de venta queda para una spec futura.
