# Feature Specification: Corrección de zona horaria en el POS de staff (previsualización de promociones) (A-09)

**Feature Branch**: `023-correccion-zona-horaria-pos-staff`

**Created**: 2026-08-18

**Status**: Draft

**Input**: User description: "Especificación delta: el POS de staff (`pos-heladeria`,
`pos-terminal.store.ts`) previsualiza la vigencia de promociones con el reloj del
dispositivo/navegador (`new Date()`, cuatro sitios: líneas 248, 262, 386 y 1190) en vez de la
hora del servidor convertida a `TENANT_TIMEZONE`, porque `promotion-pricing.util.ts` nunca recibe
esa hora en ningún endpoint que consuma. Corrige la anomalía A-09 de `registro-de-anomalias.md`,
reabriendo su tratamiento cerrado ('PENDIENTE, mitigado operativamente — relojes de terminal
verificados, documentar sin especificar') para pasar a corrección real, mismo criterio ya aplicado
a A-08 (spec 022). No toca el motor de promociones del backend (protegido, A-07) ni el criterio de
desempate del frontend (A-10, anomalía distinta)."

**Naturaleza de esta spec**: **delta de modernización**, no característica nueva. Corrige el
`FR-040` de la spec [012-motor-de-evaluacion-de-promociones-y-combos](../012-motor-de-evaluacion-de-promociones-y-combos/spec.md)
(User Story 8), que documentó A-09 explícitamente como "PENDIENTE, mitigado operativamente...
documenta el comportamiento observado sin fijarlo como contrato deseado ni obligatorio para la
modernización". No modifica el motor de promociones en sí (`active_discount_promotions`,
`local_now`, `best_line_discount` — spec 012, dueña de A-07, **protegida, no se toca**) ni el
criterio de desempate del frontend entre promociones empatadas (A-10, User Story 9 de la misma
spec — anomalía distinta, sin cambio).

**Autorización de negocio (Principio I de la [Constitución](../../.specify/memory/constitution.md))**:
`registro-de-anomalias.md`, entrada A-09, clasificada `PENDIENTE`, tratamiento previo "mitigado
operativamente" (respuesta P6 de `entrevista-negocio.md`: los relojes de los terminales del local
están verificados y fijados a `America/Bogota`) con acuerdo explícito de "documentar sin
especificar como contrato... ni obligatorio para la modernización". **Esta spec reabre esa
decisión**: el propietario del repositorio (deimerhdz21@gmail.com), actuando como negocio, decide
el 2026-08-18 corregir el defecto de diseño en vez de seguir dependiendo de la disciplina operativa
de configuración de relojes de cada terminal — el mismo criterio de riesgo latente sin corregir que
ya llevó a corregir A-08 en menú/carrito (spec 022, por contraste directo con A-07). Esta
reapertura debe quedar registrada con quién y cuándo en `registro-de-anomalias.md` como parte del
trabajo de esta spec (ver `FR-007`).

**Nota de alcance importante (verificada leyendo el código real, no solo la spec)**:
`isPromoActiveNow(promo, now)` (`promotion-pricing.util.ts:35-48`) ya recibe `now: Date` como
parámetro — es una función pura, correcta, que **no se modifica**. El defecto está enteramente en
quién construye ese `now`: `pos-terminal.store.ts` tiene **cuatro** sitios con
`const now = new Date()` (líneas 248 y 262, para combos vigentes e insignias de descuento por
producto; 386 y 1190, para el precio del carrito que ve el cajero, `discountedUnitPrice`) —
ninguno tiene acceso hoy a la hora del servidor ni a `TENANT_TIMEZONE`, porque no viaja en ningún
endpoint que el store consuma (incluido `GET /promotions`). Esta delta corrige esos cuatro puntos
de invocación para que usen una hora sincronizada con el servidor (convertida a la zona horaria del
tenant), sin tocar `promotion-pricing.util.ts` ni el motor de promociones del backend. El panel de
administración (`promotions-page.component.ts`, `getPromoDisplay`/`findOverlaps`) tiene el mismo
defecto pero es una superficie distinta (back-office, no POS de venta) — fuera de alcance.

**Test existente (Principio II)**: no existe hoy ningún test que congele el comportamiento
defectuoso de A-09 — ni en `pos-terminal.store.spec.ts` (sin tests sobre los cuatro sitios
`new Date()`) ni en `promotion-pricing.util.spec.ts` (prueba `isPromoActiveNow` solo con un `now`
explícito, inyectado por el test, nunca a través de la construcción real del store). Esta delta
debe crear characterization tests nuevos para los cuatro puntos de invocación, citando A-09.

## User Scenarios & Testing *(mandatory)*

<!--
  Igual que las specs 007/012/020/021/022 de las que depende, esta delta cita nombres de función,
  archivo y línea porque son el contrato observable que se está corrigiendo, no una fuga de
  detalles de implementación (ver Assumptions).
-->

### User Story 1 - El cajero ve la vigencia real de las promociones, sin importar el reloj del terminal (Priority: P1) — anomalía A-09

Un cajero arma un pedido en un terminal cuyo reloj/zona horaria del sistema operativo no coincide
con `America/Bogota` (tablet de fábrica en UTC, PC con detección de región fallida, terminal remoto
de soporte técnico). El POS hoy evalúa la vigencia de cada promoción con `new Date()` del
dispositivo, sin conversión — puede mostrar una promoción vigente como fuera de ventana, o
viceversa, aunque el backend (que sí evalúa correctamente en hora de Bogotá) vaya a cobrar el monto
contrario al mostrado.

**Why this priority**: es el punto de mayor exposición del defecto — el cajero arma el pedido y le
comunica un precio al cliente basado en lo que ve en pantalla, antes de que el backend confirme el
cobro real.

**Independent Test**: se puede probar de forma aislada fijando el reloj del entorno de prueba a un
instante en el que la hora UTC cae dentro de la ventana de una promoción pero la hora de Bogotá no
(o viceversa), llamando a los puntos de invocación de `pos-terminal.store.ts` (combos, insignias,
carrito) y verificando que el resultado coincide con la evaluación en hora de Bogotá, no con la
hora cruda del reloj del entorno de prueba.

**Acceptance Scenarios**:

1. **Given** una promoción con ventana horaria 17:00-19:00 hora de Bogotá y un terminal cuyo reloj
   del sistema marca UTC sin corregir, **When** el instante real es 2026-08-15 17:30 Bogotá (22:30
   UTC, dentro de ventana), **Then** el POS de staff muestra la promoción vigente (insignia de
   descuento y precio con descuento en el carrito) — corrige el comportamiento actual, que hoy la
   muestra fuera de ventana (`pos-terminal.store.ts:248,262,386,1190`).
2. **Given** la misma promoción y terminal, **When** el instante real es 2026-08-15 15:00 Bogotá
   (20:00 UTC, fuera de ventana), **Then** el POS de staff muestra la promoción NO vigente — mismo
   resultado que produce hoy el caso correcto, sin regresión.

---

### User Story 2 - El precio del carrito que ve el cajero coincide con lo que el sistema cobra al confirmar (Priority: P1) — anomalía A-09

El mismo defecto de zona horaria afecta el precio mostrado en el carrito
(`discountedUnitPrice`, calculado en `pos-terminal.store.ts:386` y `:1190`). El cajero puede ver un
precio con o sin descuento que, al confirmar el pago, el backend (ya correcto, vía A-07) cobra de
forma distinta.

**Why this priority**: mismo peso que Historia 1 — es la manifestación económica directa del
defecto: la diferencia de $1.600 por unidad documentada en `contradiccion-01-motor-promociones-
frontend-backend.md §3.1` para el ejemplo de referencia.

**Independent Test**: se puede probar de forma aislada fijando el reloj igual que en Historia 1 y
comparando el `discountedUnitPrice` calculado por el store contra el monto que produce
`promotions.evaluate` en el backend para el mismo instante y la misma promoción.

**Acceptance Scenarios**:

1. **Given** la misma promoción de Historia 1 y un ítem elegible en el carrito, **When** se fija el
   reloj del terminal a un instante donde UTC y hora Bogotá caen en lados opuestos de la ventana,
   **Then** `discountedUnitPrice` del carrito coincide exactamente con el monto que el backend
   cobraría para ese mismo instante — corrige la divergencia actual.
2. **Given** el mismo carrito, **When** el reloj del terminal está correctamente configurado a
   `America/Bogota`, **Then** el comportamiento es idéntico al actual (sin regresión).

---

### User Story 3 - La corrección no toca el motor de promociones ni introduce bloqueos si el servidor no responde (Priority: P1)

El equipo de modernización necesita la certeza de que esta corrección (a) no cambia el cálculo real
del backend ni el criterio de desempate del frontend (A-10), y (b) no deja el POS de staff sin
poder mostrar el carrito si, momentáneamente, no hay una hora de servidor disponible (arranque en
frío, corte de red).

**Why this priority**: sin esta garantía, la corrección arriesga romper la venta en el terminal
—un riesgo mayor que el defecto que corrige— o invadir el alcance protegido del motor de
promociones (Principio III, un módulo a la vez).

**Independent Test**: se puede probar de forma aislada (a) comparando, antes y después de la
corrección, el resultado del desempate entre promociones empatadas con los mismos datos — debe ser
idéntico; y (b) simulando que el terminal arranca sin haber recibido aún ninguna respuesta del
backend y verificando que el carrito se muestra igual (sin insignias de promoción, sin bloquear la
pantalla) en vez de fallar.

**Acceptance Scenarios**:

1. **Given** dos promociones empatadas en `priority` y monto, **When** se compara el resultado de
   `bestProductDiscount` antes y después de esta corrección con los mismos datos, **Then** el
   resultado es idéntico — el criterio de desempate (A-10) no cambia.
2. **Given** un terminal que aún no ha recibido ninguna respuesta del backend, **When** el cajero
   abre el POS y arma un pedido, **Then** el carrito se muestra sin bloquearse (sin insignias de
   promoción activa hasta que llegue la primera hora de servidor) — degrada explícitamente, no
   falla en silencio ni congela la pantalla.
3. **Given** una promoción ya aplicada a un pedido o factura antes de esta corrección, **When** se
   consulta ese pedido/factura después de aplicar la corrección, **Then** su valor no cambia — sin
   recálculo retroactivo.

---

### Edge Cases

- ¿Qué pasa si el terminal nunca logra una respuesta del backend desde el arranque (sin conexión
  desde el inicio)? Ver Historia 3, Escenario 2 — degrada explícitamente, no bloquea la venta.
- ¿Qué pasa si la hora del servidor y la hora local del dispositivo divergen solo por el desfase
  normal de red (unos pocos segundos), no por una zona horaria mal configurada? No debe producir
  parpadeo visible de insignias entre vigente/no vigente; el margen de tolerancia exacto es una
  decisión de la fase de planeación (`plan.md`), no de esta spec.
- ¿Qué pasa en el instante exacto del límite de una ventana horaria (el minuto exacto de apertura o
  cierre)? Sin cambio de esta delta — el criterio de comparación (`<=` vs `<`) ya está definido por
  `isPromoActiveNow`/`inTimeWindow` (no se tocan) y por el motor de promociones (spec 012).
- ¿Qué pasa con el criterio de desempate entre promociones empatadas (A-10)? Sin cambio — ver
  Historia 3, Escenario 1.
- ¿Qué pasa con el panel de administración de promociones (`promotions-page.component.ts`)? Fuera
  de alcance — ver Out of Scope.
- ¿Qué pasa con pedidos o facturas ya generados con el precio previsualizado incorrecto antes de
  esta corrección? Quedan igual — ver Historia 3, Escenario 3; el monto ya cobrado en esos casos
  nunca estuvo afectado (el backend ya usa la hora correcta desde A-07).

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001 [Corrige A-09, `FR-040` de la spec 012, Historia 1]**: el POS de staff DEBE evaluar la
  vigencia de cada promoción (combos, insignias de descuento por producto) usando una hora
  sincronizada con el servidor y convertida a la zona horaria del tenant, no la hora cruda del
  reloj del dispositivo/navegador.
- **FR-002 [Corrige A-09, Historia 2]**: el precio previsualizado en el carrito
  (`discountedUnitPrice`, `pos-terminal.store.ts:386,1190`) DEBE calcularse con el mismo criterio de
  hora que FR-001, de modo que coincida con el monto que el backend cobraría para el mismo instante.
- **FR-003 [Historia 1/2, consistencia]**: para un mismo instante y una misma promoción, el
  resultado de vigencia mostrado en el POS de staff (insignias, precio del carrito) y el que
  efectivamente se aplica al cobrar (ya corregido por A-07) DEBE ser idéntico, sin importar el
  reloj/zona horaria configurados en el dispositivo.
- **FR-004 [Historia 3, disponibilidad]**: si el POS de staff aún no dispone de una hora
  sincronizada con el servidor (por ejemplo, arranque en frío antes de la primera respuesta del
  backend), el sistema DEBE seguir mostrando la pantalla del carrito sin bloquearse, degradando de
  forma explícita y detectable (por ejemplo, sin insignias de promoción) en vez de fallar en
  silencio o usar la hora del dispositivo sin corregir.
- **FR-005 [Sin cambio, se conserva — Historia 3]**: el criterio de desempate entre promociones
  empatadas del POS de staff (`bestProductDiscount`, anomalía A-10) NO DEBE cambiar como resultado
  de esta corrección.
- **FR-006 [Principio V de la constitución, ningún cambio retroactivo, Historia 3]**: el sistema NO
  DEBE recalcular, revertir ni alterar ninguna promoción ya aplicada, ningún pedido ni ninguna
  factura ya generados antes de esta corrección.
- **FR-007 [Principio I de la constitución, reapertura de decisión]**: la implementación DEBE
  registrar en `registro-de-anomalias.md` la reapertura del tratamiento de A-09 (de "mitigado
  operativamente, documentar sin especificar" a "corregido en modernización"), citando quién y
  cuándo tomó la decisión, igual que las demás correcciones de esta serie (A-04, A-08, A-44).
- **FR-008 [Principio II de la constitución, trazabilidad de la corrección]**: la implementación
  DEBE incluir al menos un test de characterization por cada uno de los cuatro puntos de invocación
  corregidos (`pos-terminal.store.ts:248,262,386,1190`) que demuestre, antes de la corrección, el
  comportamiento defectuoso (vigencia evaluada con el reloj del dispositivo) y, después, el
  comportamiento corregido — citando la anomalía A-09.

### Key Entities

- **Promotion**: la promoción cuya vigencia se evalúa; su ventana horaria (`start_time`/`end_time`)
  y estado (`active`) no cambian — solo la hora que se le compara en el POS de staff.
- **POS de staff (`pos-terminal.store.ts`)**: la superficie donde se muestra el resultado de esa
  evaluación antes del cobro; no persiste el resultado, se recalcula en cada render del carrito.
- **Hora sincronizada con el servidor**: el nuevo dato que el POS de staff debe obtener y mantener
  para reemplazar el reloj crudo del dispositivo en los cuatro puntos de invocación — su mecanismo
  exacto de obtención y actualización es una decisión de la fase de planeación (`plan.md`), no de
  esta spec.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: El 100% de las evaluaciones de vigencia en el POS de staff (combos, insignias,
  carrito) para un terminal con reloj/zona horaria distinta a `America/Bogota` produce el mismo
  resultado que un terminal correctamente configurado, para el mismo instante real.
- **SC-002**: El 100% de los precios previsualizados en el carrito del POS de staff coincide con el
  monto que el backend cobraría para el mismo instante y la misma promoción — sin divergencia entre
  previsualización y cobro real.
- **SC-003**: El resultado del desempate entre promociones empatadas (A-10) es idéntico antes y
  después de esta corrección, verificado con los mismos datos de entrada.
- **SC-004**: El POS de staff nunca bloquea la pantalla del carrito por falta de hora sincronizada
  con el servidor — en el peor caso, degrada sin insignias de promoción hasta obtenerla.
- **SC-005**: Ninguna promoción, pedido ni factura ya generados antes de esta corrección cambia de
  valor como consecuencia de este cambio.
- **SC-006**: Existen al menos cuatro scripts de characterization (uno por punto de invocación) que
  demuestran la divergencia previa y su cierre tras la corrección, citando A-09.

## Out of Scope

- **El motor de promociones del backend en sí** (`active_discount_promotions`, `local_now`,
  `best_line_discount`, prioridad entre promociones empatadas) — spec 012, anomalía A-07,
  **protegida, no se toca**.
- **La función pura `isPromoActiveNow`/`inTimeWindow`/`bestProductDiscount` de
  `promotion-pricing.util.ts`** — ya son correctas dado un `now` válido; el defecto está en quién
  construye ese `now`, no en estas funciones.
- **El criterio de desempate del frontend entre promociones empatadas** (A-10) — anomalía distinta,
  ver FR-005.
- **El panel de administración de promociones** (`promotions-page.component.ts`, `getPromoDisplay`,
  `findOverlaps`) — misma clase de defecto, pero superficie distinta (back-office, no POS de
  venta); no forma parte de esta delta.
- **Los cuatro caminos de cobro real** (mostrador, unificado, dividido, `pay_order` legado) — ya
  corregidos por A-07, sin cambio.
- **Sincronización de reloj a nivel de sistema operativo del dispositivo** (NTP, políticas de MDM)
  — esta corrección elimina la dependencia de esa disciplina para el cálculo de vigencia, no la
  reemplaza por otra externa ni la gestiona.
- **Auditar si el desfase llegó a afectar una venta real en producción** antes de esta corrección —
  decisión de negocio no bloqueante, igual que ya lo dejó A-07 para su propio caso histórico.

## Assumptions

- **Esta es una spec delta de corrección, no de característica nueva**: al igual que las specs
  007/012/020/021/022, cita nombres de función, archivo y línea explícitamente porque son el
  contrato observable que se está corrigiendo — los criterios de aceptación deben poder verificarse
  directamente contra `pos-heladeria` en ejecución, antes y después del cambio.
- **La reapertura de la decisión de negocio de A-09 se documenta como parte de esta spec, no como
  una nueva ronda de entrevista real**: el propietario del repositorio decide corregir el defecto de
  diseño en vez de mantener la mitigación operativa, siguiendo el mismo criterio de riesgo latente
  ya aplicado a A-08 (ver FR-007). No se requiere una nueva entrevista de negocio porque el defecto
  técnico y su tratamiento alternativo ya están completamente caracterizados en
  `contradiccion-01-motor-promociones-frontend-backend.md`.
- **El mecanismo exacto para obtener y mantener la "hora sincronizada con el servidor" en el POS de
  staff es una decisión de diseño de la fase de planeación (`plan.md`)**, no de esta spec — esta
  spec solo exige que exista tal mecanismo y que reemplace el reloj crudo del dispositivo en los
  cuatro puntos de invocación citados (FR-001/FR-002).
- **No se requiere migración de datos**: el cambio es puramente de qué hora se usa para evaluar
  vigencia en tiempo de lectura en el cliente; no hay estado persistido que migrar.
- **Los nombres de función, archivo y línea citados son literales del código al momento de esta
  extracción** (2026-08-18, `pos-heladeria/src/app/modules/tables/services/pos-terminal.store.ts`,
  `pos-heladeria/src/app/modules/promotions/services/promotion-pricing.util.ts`). Si el código
  cambia, esta spec debe re-verificarse antes de usarse como characterization test.
