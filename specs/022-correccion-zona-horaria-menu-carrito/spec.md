# Feature Specification: Corrección de zona horaria en vigencia de promociones del menú y carrito QR (A-08)

**Feature Branch**: `022-correccion-zona-horaria-menu-carrito`

**Created**: 2026-08-18

**Status**: Draft

**Input**: User description: "Especificación delta: corrige la evaluación de vigencia de
promociones en `menu/router.py:82` (`_build_menu`) y `cart/service.py:205` (`serialize_cart`) para
que pasen un `datetime` aware (`datetime.now(timezone.utc)`) a
`promotions.active_discount_promotions`, en vez del `datetime` naive derivado de UTC que hoy
`promotions.local_now()` interpreta incorrectamente como si ya fuera hora local del tenant. Cierra
la anomalía A-08 de `registro-de-anomalias.md` (ACCIDENTAL, por contraste directo con A-07, que sí
se corrigió en los cuatro caminos de cobro real). No toca `_now()` de `cart/service.py` en sí (usada
también para `expires_at` de la sesión del comensal, anomalía distinta) ni el motor de promociones
(`active_discount_promotions`/`local_now`/`best_line_discount`, spec 012, dueña de A-07,
protegida)."

**Naturaleza de esta spec**: **delta de modernización**, no característica nueva. Corrige el
`FR-030` de la spec [007-menu-carrito-qr](../007-menu-carrito-qr/spec.md), que documentó el
comportamiento actual (UTC naive tratado como hora local) tal cual, sin especificarlo como
corrección obligatoria ("Tratamiento acordado: corregirlo en modernización aplicando el patrón ya
usado en los caminos de cobro real"). No modifica el motor de promociones en sí
(`active_discount_promotions`, `local_now`, `best_line_discount` — spec
[012-motor-de-evaluacion-de-promociones-y-combos](../012-motor-de-evaluacion-de-promociones-y-combos/spec.md),
dueña de A-07, **protegida, no se toca**), solo los dos puntos de invocación que le pasan un
`datetime` mal construido.

**Autorización de negocio (Principio I de la [Constitución](../../.specify/memory/constitution.md))**:
`registro-de-anomalias.md`, entrada A-08, clasificada **ACCIDENTAL** por contraste directo con A-07
(que sí se corrigió en los cuatro caminos de cobro real — CÓDIGO por sí solo basta para ACCIDENTAL).
"Tratamiento acordado": corregir en fase de modernización aplicando exactamente el mismo patrón que
ya funciona en `checkout.py`/`table_sessions/service.py`/`sales/service.py` (pasar un `datetime`
aware, `datetime.now(timezone.utc)` sin `.replace(tzinfo=None)`). "Decisión de negocio pendiente":
ninguna — el propio registro lo marca como "corrección técnica directa, no una decisión de
producto".

**Nota de alcance importante (verificada leyendo el código real, no solo la spec)**:
`cart/service.py` tiene una función compartida `_now()` (líneas 52-53) usada en **dos** sitios: (1)
línea 107, para calcular `expires_at` de la sesión del comensal (TTL deslizante, un `DateTime`
naive en base de datos, comparado en `qr_context.py`/`scheduler.py` con `datetime.now(
timezone.utc).replace(tzinfo=None)`, también naive); y (2) línea 205, dentro de `serialize_cart`,
pasado a `promotions.active_discount_promotions` — el único uso relacionado con A-08. Convertir
`_now()` en sí a timezone-aware rompería la comparación de `expires_at` (Principio III, un módulo a
la vez: la expiración de sesión es una anomalía distinta, no A-08). Por eso esta delta corrige el
`datetime` **solo en el punto de invocación de `serialize_cart`** (línea 205) y en `menu/router.py`
(línea 82, dentro de `_build_menu`, donde `now` no se usa para nada más que para las promociones) —
sin tocar `_now()` ni `expires_at`.

**Test existente que esta delta debe actualizar (Principio II)**: la spec
[015-caracterizacion-cart](../015-caracterizacion-cart/spec.md) ya congeló A-08 explícitamente en
`test_open_session_y_serialize_cart_a08_zona_horaria_no_aplicada`
(`app/characterization_tests/test_cart_service.py`), con la advertencia explícita de que esa spec
"NO DEBE corregir, mitigar ni alterar A-08" (`FR-011`). Esta delta es la que sí corrige — el
Principio II exige modificar ese test citando A-08 en el commit, no "ajustarlo" en silencio. No
existe hoy un test equivalente para `menu/router.py`; esta delta debe crear uno.

## User Scenarios & Testing *(mandatory)*

<!--
  Igual que las specs 007/015/020/021 de las que depende, esta delta cita nombres de función,
  archivo y línea porque son el contrato observable que se está corrigiendo, no una fuga de
  detalles de implementación (ver Assumptions).
-->

### User Story 1 - El menú público muestra la vigencia real de una promoción (Priority: P1) — anomalía A-08

Un comensal abre el menú público por QR cerca del límite horario de una promoción. El backend hoy
evalúa esa vigencia con un `datetime` naive derivado de UTC que `promotions.local_now()`
interpreta incorrectamente como si ya fuera hora local del tenant — con `TENANT_TIMEZONE=
America/Bogota` (UTC-5), esto puede mostrar la promoción vigente (o no vigente) hasta 5 horas antes
o después de su ventana real.

**Why this priority**: es el punto de mayor exposición — el menú es lo primero que ve cualquier
comensal, y un desajuste ahí es la fuente más directa de reclamos cuando el precio cobrado
(correcto, vía A-07) no coincide con lo que vio antes de pedir.

**Independent Test**: se puede probar de forma aislada llamando `_build_menu` (o `GET /menu`) con
el reloj fijado a un instante en el que la hora UTC cae dentro de la ventana de una promoción pero
la hora de Bogotá no, y verificando que la promoción aparece como NO vigente.

**Acceptance Scenarios**:

1. **Given** una promoción con ventana horaria 20:00-21:00 hora de Bogotá (UTC-5), **When** se
   consulta el menú público a las 20:00 UTC (15:00 Bogotá, fuera de ventana), **Then** la promoción
   aparece como NO vigente — corrige el comportamiento actual, que hoy la muestra vigente
   (`menu/router.py:82`, `_build_menu`).
2. **Given** la misma promoción, **When** se consulta el menú público a las 01:00 UTC del día
   siguiente (20:00 Bogotá, dentro de ventana), **Then** la promoción aparece vigente — mismo
   resultado que produce hoy el caso correcto, sin regresión.

---

### User Story 2 - El carrito del comensal muestra el mismo estado de vigencia que el menú y el cobro (Priority: P1) — anomalía A-08

El mismo defecto de zona horaria ocurre en `serialize_cart` (`cart/service.py:205`) al calcular el
`discounted_total` del carrito. El comensal puede ver una promoción aplicada en su carrito que, al
pagar, ya no está vigente según los caminos de cobro real (ya corregidos por A-07) — o viceversa.

**Why this priority**: mismo peso que Historia 1 — es el segundo punto exacto que cita A-08, y sin
corregirlo el carrito seguiría divergiendo del menú y del cobro real aunque el menú ya esté
corregido.

**Independent Test**: se puede probar de forma aislada llamando `serialize_cart` con el reloj
fijado igual que en Historia 1 y verificando que `discounted_total` no aplica el descuento fuera de
la ventana real.

**Acceptance Scenarios**:

1. **Given** la misma promoción de Historia 1 y un carrito con un ítem elegible, **When** se
   fija el reloj a las 20:00 UTC (15:00 Bogotá, fuera de ventana) y se llama `serialize_cart`,
   **Then** `discounted_total` es igual a `total` (sin descuento) — corrige el comportamiento
   actual, que hoy sí descuenta.
2. **Given** el mismo carrito, **When** se fija el reloj a las 01:00 UTC del día siguiente (20:00
   Bogotá, dentro de ventana), **Then** `discounted_total` refleja el descuento — mismo resultado
   que produce hoy el caso correcto, sin regresión.

---

### User Story 3 - La corrección no toca nada fuera de la evaluación de promociones (Priority: P1)

El equipo de modernización necesita la certeza de que esta corrección no cambia el vencimiento de
la sesión del comensal por inactividad (`expires_at`, calculado por la misma función `_now()` que
usa `serialize_cart`), ni ningún otro comportamiento fuera de la vigencia de promociones.

**Why this priority**: sin esta garantía, la corrección arriesga romper el TTL de sesión del
comensal —una anomalía completamente distinta— violando el Principio III (un módulo a la vez) y
generando una regresión no relacionada con A-08.

**Independent Test**: se puede probar de forma aislada comparando, antes y después de la
corrección, el instante exacto en que expira una sesión de comensal con el reloj fijado — debe ser
idéntico.

**Acceptance Scenarios**:

1. **Given** una sesión de comensal con `expires_at` calculado, **When** se compara ese valor antes
   y después de aplicar esta corrección con el mismo reloj fijado, **Then** el valor es idéntico —
   `_now()` y su uso en `open_session` (línea 107) no cambian.
2. **Given** una promoción ya aplicada a un pedido o factura antes de esta corrección, **When** se
   consulta ese pedido/factura después de aplicar la corrección, **Then** su valor no cambia — sin
   recálculo retroactivo.

---

### Edge Cases

- ¿Qué pasa exactamente en el límite de la ventana horaria (el minuto exacto de apertura o cierre)?
  Sin cambio de esta delta — el criterio de comparación (`<=` vs `<`) ya está definido por el motor
  de promociones (spec 012) y no se toca aquí.
- ¿Qué pasa si el reloj del servidor no está en UTC? Fuera de alcance — el sistema ya asume reloj
  del servidor en UTC en todos los caminos correctos (checkout, mesas, ventas); esta delta solo
  iguala el menú/carrito a esa misma asunción, no la introduce.
- ¿Qué pasa con la sesión del comensal (`expires_at`) y el barrido de sesiones huérfanas
  (`scheduler.py`)? Sin cambio — ver Historia 3; esta corrección no toca `_now()` ni ningún cálculo
  de expiración.
- ¿Qué pasa si no hay ninguna promoción activa? Sin cambio frente a hoy — el menú/carrito siguen
  mostrando precios normales; la corrección solo afecta el criterio de vigencia cuando sí existe una
  promoción con ventana horaria.
- ¿Qué pasa con los pedidos o facturas ya generados con el estado de vigencia incorrecto mostrado
  antes de esta corrección? Quedan igual — ver Historia 3, sin recálculo retroactivo; además, el
  monto ya cobrado en esos casos nunca estuvo afectado (los caminos de cobro real ya usan la hora
  correcta desde A-07).

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001 [Corrige A-08, `FR-030` de la spec 007, Historia 1]**: `_build_menu`
  (`menu/router.py:82`) DEBE calcular la vigencia de promociones pasando a
  `active_discount_promotions` un `datetime` aware en UTC (`datetime.now(timezone.utc)`, sin
  `.replace(tzinfo=None)`), igual que ya hacen los caminos de cobro real.
- **FR-002 [Corrige A-08, `FR-030` de la spec 007, Historia 2]**: `serialize_cart`
  (`cart/service.py:205`) DEBE calcular la vigencia de promociones pasando a
  `active_discount_promotions` un `datetime` aware en UTC, con el mismo criterio que FR-001 — sin
  modificar la función compartida `_now()` (líneas 52-53) ni su uso en `open_session` (línea 107).
- **FR-003 [Historia 1/2, consistencia]**: para un mismo instante y una misma promoción, el
  resultado de vigencia del menú (`_build_menu`), del carrito (`serialize_cart`) y el que
  efectivamente se aplica al cobrar (ya corregido por A-07) DEBE ser idéntico.
- **FR-004 [Sin cambio, se conserva — Historia 3]**: el cálculo de `expires_at` de la sesión del
  comensal (`open_session`, `cart/service.py:107`, vía `_now()`) NO DEBE cambiar como resultado de
  esta corrección.
- **FR-005 [Principio V de la constitución, ningún cambio retroactivo, Historia 3]**: el sistema NO
  DEBE recalcular, revertir ni alterar ninguna promoción ya aplicada, ningún pedido ni ninguna
  factura ya generados antes de esta corrección.
- **FR-006 [Principio II de la constitución, trazabilidad de la corrección]**: la implementación
  DEBE incluir al menos un test de characterization por cada uno de los dos puntos de invocación
  corregidos (menú y carrito) que demuestre, antes de la corrección, el comportamiento defectuoso
  y, después, el comportamiento corregido — citando la anomalía A-08. El test existente
  `test_open_session_y_serialize_cart_a08_zona_horaria_no_aplicada`
  (`app/characterization_tests/test_cart_service.py`, de la spec 015) DEBE actualizarse citando
  A-08 en el commit (Principio II), no reemplazarse en silencio.

### Key Entities

- **Promotion**: la promoción cuya vigencia se evalúa; su ventana horaria (`start_time`/`end_time`)
  y estado (`active`) no cambian — solo el `datetime` que se le compara.
- **Menú público / Carrito**: las dos superficies de lectura donde se muestra el resultado de esa
  evaluación antes del cobro; ninguna persiste el resultado, se recalcula en cada consulta.
- **SessionParticipant**: entidad cuyo campo `expires_at` usa la misma función `_now()` que
  `serialize_cart` en un punto distinto — se documenta aquí explícitamente porque su exclusión del
  alcance (FR-004) es tan relevante como la corrección misma.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: El 100% de las consultas al menú público con una promoción cuya ventana horaria en
  hora de Bogotá difiere de la ventana en UTC muestra el estado de vigencia correcto en hora local,
  sin excepción.
- **SC-002**: El 100% de las llamadas a `serialize_cart` en el mismo escenario muestra el mismo
  estado de vigencia que el menú y que el cobro real — sin divergencia entre los tres.
- **SC-003**: El instante exacto de expiración de una sesión de comensal (`expires_at`) es idéntico
  antes y después de esta corrección, verificado con reloj fijado.
- **SC-004**: Ninguna promoción, pedido ni factura ya generados antes de esta corrección cambia de
  valor como consecuencia de este cambio.
- **SC-005**: Existen al menos dos scripts de characterization (uno por punto de invocación) que
  demuestran la divergencia previa y su cierre tras la corrección, citando A-08.

## Out of Scope

- **El motor de promociones en sí** (`active_discount_promotions`, `local_now`,
  `best_line_discount`, prioridad entre promociones empatadas) — spec 012, anomalía A-07,
  **protegida, no se toca**.
- **El vencimiento de la sesión del comensal por inactividad** (`expires_at`, `_now()` en su uso de
  `open_session`) — ver FR-004/Edge Cases, anomalía distinta, no A-08.
- **Los cuatro caminos de cobro real** (mostrador, unificado, dividido, `pay_order` legado) — ya
  corregidos por A-07, sin cambio.
- **El resto de anomalías documentadas en las specs 007 y 015** (A-17/R16, A-21, A-24, A-28, A-36,
  A-47) — defectos distintos, no forman parte de esta delta.
- **Cualquier cambio de UI/frontend** — el menú/carrito ya consumen los mismos endpoints; solo
  cambia el criterio interno de vigencia que ya deberían haber aplicado.
- **Auditar si el desfase llegó a afectar una promoción real mostrada a un comensal en producción**
  antes de esta corrección — decisión de negocio no bloqueante, igual que ya lo dejó A-07 para su
  propio caso histórico.

## Assumptions

- **Esta es una spec delta de corrección, no de característica nueva**: al igual que las specs
  007/015/020/021, cita nombres de función, archivo y línea explícitamente porque son el contrato
  observable que se está corrigiendo — los criterios de aceptación deben poder verificarse
  directamente contra `pos-backend` en ejecución, antes y después del cambio.
- **La autorización de negocio ya existe y no requiere una nueva ronda de entrevista**: el
  "Tratamiento acordado" de A-08 en `registro-de-anomalias.md`, clasificado ACCIDENTAL por
  contraste directo con A-07, satisface el Principio I de la constitución sin necesidad de
  testimonio adicional de negocio ("Decisión de negocio pendiente: ninguna").
- **No se toca `_now()` de `cart/service.py`**: se corrige el `datetime` únicamente en el punto de
  invocación de `serialize_cart` (línea 205) y en `_build_menu` de `menu/router.py` (línea 82),
  dejando intacta la función compartida y su uso para `expires_at` — decisión de alcance explícita
  para no introducir una regresión en la expiración de sesión, ajena a A-08 (Principio III).
- **No se requiere migración de datos**: el cambio es puramente de qué `datetime` se pasa a una
  función de evaluación en tiempo de lectura; no hay estado persistido que migrar.
- **Los nombres de función y línea citados son literales del código al momento de esta extracción**
  (2026-08-18, `pos-backend/app/api/v1/cart/service.py`, `app/api/v1/menu/router.py`). Si el código
  cambia, esta spec debe re-verificarse antes de usarse como characterization test.
