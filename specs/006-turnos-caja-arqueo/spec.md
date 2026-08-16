# Feature Specification: Turnos de caja, movimientos manuales y arqueo

**Feature Branch**: `006-turnos-caja-arqueo`

**Created**: 2026-08-16

**Status**: Draft

**Naturaleza de esta spec**: **ingeniería inversa / characterization spec**. No describe una
funcionalidad nueva: documenta el comportamiento que el sistema **ya tiene hoy** en
`pos-backend/app/api/v1/cash/service.py` y `router.py` (apertura y cierre de turno, movimientos
manuales de efectivo, arqueo parcial e histórico), para que sirva de contrato formal de cara a la
modernización (Principio I y Principio III de la
[Constitución](../../.specify/memory/constitution.md)), tomando `reglas-de-negocio.md`
(RN-CASH-01 a RN-CASH-17) y `registro-de-anomalias.md` (A-20, A-17, A-40) como fuente.
**Excepción explícita a la naturaleza de "solo documentar"**: dos de las diecisiete reglas
(RN-CASH-13 y RN-CASH-17, ambas dentro de la anomalía A-20) no se especifican tal cual existen
hoy — el negocio, al ser consultado, confirmó requisitos **más estrictos** que el comportamiento
actual, así que esta spec fija el contrato **deseado y ya confirmado**, no el vigente, para esas
dos reglas. El resto (RN-CASH-01 a 12, 14 a 16) documenta el comportamiento existente sin
cambios. Ver la nota de alcance en cada User Story afectada y en Assumptions.

**Input**: User description: "Spec de ingeniería inversa: documenta el comportamiento EXISTENTE de
los turnos de caja, los movimientos manuales de efectivo y el arqueo del sistema POS Heladería. No
es una feature nueva: es la especificación formal de lo que el sistema YA hace, para que sirva de
contrato en la modernización. Alcance — RN-CASH-01 a RN-CASH-17,
`pos-backend/app/api/v1/cash/service.py`, `router.py`. Incluye las anomalías A-20 (tres reglas:
RN-CASH-09 mitigada por pantalla, RN-CASH-13 con requisito de negocio confirmado en ronda 2, y
RN-CASH-17 con requisito nuevo de snapshot inmutable + vista de ajustes separada, surgido en la
entrevista P14), A-17 (falta de lock de fila en `close_shift`/`add_movement`, ACCIDENTAL, corregir)
y A-40 (alias `cash_sales` deprecado, PENDIENTE confirmar migración del frontend). Sin
characterization test dedicado entre los 12 scripts existentes. Fuera de alcance: cómo se generan
las ventas que el arqueo agrega (spec 011) y el cierre de cuenta de mesa en sí (spec 010)."

## User Scenarios & Testing *(mandatory)*

<!--
  Cada escenario documenta un comportamiento OBSERVADO en `cash/service.py` y `cash/router.py`,
  salvo las User Stories 6 y 7, marcadas explícitamente como REQUISITO NUEVO CONFIRMADO — ahí el
  escenario documenta el contrato deseado, con el comportamiento actual citado aparte como el gap
  que ese contrato cierra. Las anomalías conocidas se marcan inline con su tratamiento acordado
  (`registro-de-anomalias.md`).
-->

### User Story 1 - Un solo turno de caja abierto por caja registradora a la vez (Priority: P1)

Un cajero abre turno en una caja registradora al empezar su jornada, declarando el fondo inicial
en efectivo. Si esa caja ya tiene un turno abierto (propio o de otro cajero que no cerró), el
sistema rechaza el intento en vez de crear un segundo turno concurrente.

**Why this priority**: es la garantía de integridad más básica del módulo — sin ella, dos turnos
abiertos a la vez sobre la misma caja harían ambiguo a cuál turno pertenece cada venta y cada
movimiento manual, invalidando cualquier arqueo posterior.

**Independent Test**: se puede probar con `POST /cash/shifts/open` dos veces seguidas con el mismo
`cash_register_id`, sin cerrar el primero.

**Acceptance Scenarios**:

1. **Given** una caja registradora sin turno abierto, **When** un cajero abre turno con
   `opening_amount=50.000`, **Then** el sistema crea el turno en estado `open` y lo asocia al
   usuario autenticado (`RN-CASH-01`).
2. **Given** esa misma caja con el turno recién abierto, **When** se intenta abrir un segundo
   turno para la misma `cash_register_id`, **Then** el sistema rechaza con `409` "La caja ya
   tiene un turno abierto" — la restricción es de base de datos (índice único parcial sobre
   turnos `open`), no solo una validación de aplicación (`RN-CASH-01`).
3. **Given** una caja registradora que no existe, **When** se intenta abrir turno sobre ella,
   **Then** el sistema rechaza con `404` antes de evaluar la restricción de turno único.

---

### User Story 2 - El efectivo esperado del cajón se calcula en tiempo real a partir de pagos reales (Priority: P1)

En cualquier momento del turno (o al cerrarlo), el sistema calcula cuánto efectivo debería haber
en el cajón. Ese cálculo nunca lee un registro propio de "ventas del turno": suma, en el momento
de la consulta, los pagos reales (`Payment`) de las ventas (`Sale`) asociadas al turno y ya
pagadas, filtra solo los de tipo efectivo, les resta el cambio entregado, y le suma o resta los
movimientos manuales según su tipo.

**Why this priority**: es el corazón del módulo — todas las demás reglas (desglose, cierre,
arqueo parcial, histórico) parten de esta fórmula. Un error aquí invalida cualquier arqueo,
incluida la confianza de la gestoría en el efectivo reportado.

**Independent Test**: se puede probar invocando `service.reconcile(db, shift)` sobre un turno con
ventas mixtas (efectivo, tarjeta, transferencia), alguna venta anulada, cambio entregado y
movimientos manuales de los tres tipos, y verificando cada componente de la fórmula por separado.

**Acceptance Scenarios**:

1. **Given** un turno con una venta de $50.000 pagada en efectivo y otra de $30.000 pagada con
   tarjeta, **Then** el efectivo esperado solo refleja la de $50.000 — el sistema nunca guarda ni
   lee un registro propio en `cash_movements` para las ventas; las deriva de `Sale`/`Payment` en
   cada consulta (`RN-CASH-02`).
2. **Given** ese mismo turno, **When** una de las dos ventas se anula (`status` distinto de
   `paid`) después de haberse registrado, **Then** el cálculo del efectivo esperado **no** la
   cuenta, aunque tenga pagos asociados — solo cuentan ventas con `status='paid'` (`RN-CASH-02`).
3. **Given** pagos en efectivo, con tarjeta y por transferencia dentro del mismo turno, **Then**
   únicamente el de tipo efectivo (`PaymentMethod.type='cash'`) suma o resta al efectivo esperado
   del cajón; tarjeta y transferencia se reportan en el desglose sin afectar ese monto
   (`RN-CASH-03`).
4. **Given** pagos en efectivo que suman $120.000 brutos y un total de cambio entregado de
   $15.000 en las ventas del turno, **Then** el "efectivo esperado por ventas" es $105.000, no
   $120.000 — el cambio entregado sale físicamente del cajón, así que se resta antes de sumar al
   esperado (`RN-CASH-04`).
5. **Given** `opening_amount=50.000`, `ventas_efectivo=105.000` (ya neto de cambio),
   `ingresos=10.000`, `egresos=8.000`, `retiros=20.000`, **Then** el efectivo esperado total es
   exactamente `50.000 + 105.000 + 10.000 - 8.000 - 20.000 = 137.000` — la fórmula completa del
   cierre (`RN-CASH-05`).
6. **Given** un turno recién abierto que aún no registró ningún conteo físico
   (`counted_amount=None`), **When** se consulta su arqueo, **Then** la diferencia queda `None`
   explícitamente — nunca se asume `0` ni el valor esperado como si coincidiera; `None`
   distingue "no contado todavía" de "contado y coincide exacto" (`RN-CASH-06`).

---

### User Story 3 - El desglose de ventas por método de pago muestra todos los métodos activos, efectivo primero (Priority: P2)

Al consultar el arqueo de un turno, el cajero ve una lista con el total y el número de ventas de
cada método de pago activo del negocio — incluidos los métodos que no tuvieron ninguna venta ese
turno, mostrados en cero. El efectivo aparece siempre primero; el resto, alfabéticamente.

**Why this priority**: es la vista que usa el cajero para verificar visualmente el arqueo antes de
cerrar; una lista incompleta (que omita un método sin ventas) daría la falsa impresión de que ese
método no existe en el sistema.

**Independent Test**: se puede probar consultando `GET /cash/shifts/{id}/reconciliation` sobre un
turno donde solo se usó efectivo, con un método de pago "Transferencia" activo pero sin ninguna
venta ese turno.

**Acceptance Scenarios**:

1. **Given** un turno con ventas solo en efectivo y un método "Transferencia" activo sin ventas
   ese turno, **Then** "Transferencia" igual aparece en el desglose con `total=0, count=0`
   — antes de esta corrección, un `INNER JOIN` hacía que los métodos sin ventas simplemente no
   aparecieran (`RN-CASH-07`).
2. **Given** un desglose con efectivo, "Nequi" y "Tarjeta" con ventas, **Then** el orden de
   presentación es: efectivo primero siempre, y el resto ordenado alfabéticamente sin distinguir
   mayúsculas de minúsculas (`RN-CASH-08`).

---

### User Story 4 - Cierre de turno: conteo físico, observación obligatoria si no cuadra, y no se cierra dos veces (Priority: P1)

Un cajero cierra su turno al final de la jornada, reportando el conteo físico del cajón —ya sea
denominación por denominación (billete/moneda × cantidad) o como un monto único—. Si el conteo no
coincide con lo esperado, el sistema exige que el cajero deje una observación explicando la
diferencia antes de permitir el cierre. Un turno ya cerrado no puede volver a cerrarse.

**Why this priority**: es el evento que fija el resultado del turno para el resto del negocio
(reportes, gestoría); las tres guardas (fuente del conteo, observación obligatoria, no doble
cierre) son las que le dan seriedad a ese resultado.

**Independent Test**: se puede probar con `POST /cash/shifts/{id}/close` variando: enviar
denominaciones vs. un monto único vs. ninguno de los dos; provocar una diferencia distinta de
cero con y sin `close_note`; y repetir el cierre sobre un turno ya cerrado.

**Acceptance Scenarios**:

1. **Given** un cierre que envía denominaciones (p. ej. 3 billetes de $50.000 + 10 monedas de
   $1.000), **Then** el `counted_amount` se calcula como la suma de `denominación × cantidad`
   ($160.000), ignorando cualquier `counted_amount` que también venga en el mismo request
   (`RN-CASH-09`).
2. **Given** un cierre que **no** envía denominaciones pero sí un `counted_amount` directo,
   **Then** ese valor se usa tal cual, sin necesidad de detalle por denominación (`RN-CASH-09`).
3. **Given** una diferencia distinta de cero entre lo contado y lo esperado, **When** el cajero
   cierra sin `close_note` (o con una cadena vacía/solo espacios), **Then** el sistema rechaza con
   `422` "El arqueo no cuadra: la observación (close_note) es obligatoria" (`RN-CASH-10`).
4. **Given** una diferencia exactamente igual a cero, **Then** `close_note` es opcional — el
   sistema no exige nada adicional (`RN-CASH-10`).
5. **Given** un turno ya cerrado, **When** se intenta cerrarlo de nuevo, **Then** el sistema
   rechaza con `409` "El turno ya está cerrado", sin tocar ningún dato del cierre original
   (`RN-CASH-11`).

**Anomalía A-20/RN-CASH-09 — clasificación `DUDOSA`, tratamiento cerrado (ronda 1, pregunta P14,
`mitigado`)**: el backend por sí solo **permite** cerrar un turno con `counted_amount=None` (sin
denominaciones y sin monto manual) — el escenario 1/2 de arriba no cubre ese tercer caso porque no
está bloqueado a nivel de API. El negocio confirmó en la entrevista que **la pantalla de cierre
siempre exige el conteo antes de habilitar el botón de cerrar**, así que el riesgo está mitigado
en el frontend, no en el contrato del backend. Esta spec documenta el comportamiento del backend
tal cual (permisivo) y dejar constancia de la mitigación operativa — **no** se agrega aquí una
validación de backend que exija `counted_amount` no nulo, porque el negocio no la pidió; si se
detecta en el futuro un cliente distinto al frontend actual que llame a este endpoint sin esa
guarda de pantalla, esta mitigación deja de cubrir el riesgo.

---

### User Story 5 - Movimientos manuales de efectivo: tres tipos con signo fijo, monto siempre positivo (Priority: P1)

Un cajero registra un ingreso, un egreso o un retiro de efectivo del cajón durante el turno (por
ejemplo, un vuelto insuficiente que obliga a traer cambio, o un retiro de seguridad a mitad de
jornada). El tipo de movimiento (`kind`) determina si suma o resta al efectivo esperado; el monto
siempre se registra en positivo — nunca es el signo de `amount` el que indica la dirección.
Los movimientos solo pueden registrarse mientras el turno sigue abierto.

**Why this priority**: son la única otra fuente (además de las ventas) que mueve el efectivo
esperado del cajón — necesarios para que el arqueo de cierre cuadre con la realidad operativa del
día (no solo con las ventas).

**Independent Test**: se puede probar con `POST /cash/shifts/{id}/movements` enviando los tres
`kind` posibles con `amount` positivo, y verificando el efecto de cada uno en `reconcile`; y
repitiendo la llamada sobre un turno ya cerrado.

**Acceptance Scenarios**:

1. **Given** un turno abierto, **When** se registra un movimiento `kind='ingreso'` de $10.000,
   **Then** `expected` sube en $10.000 (`RN-CASH-14`, `RN-CASH-05`).
2. **Given** el mismo turno, **When** se registra un movimiento `kind='egreso'` de $8.000,
   **Then** `expected` baja en $8.000 (`RN-CASH-14`, `RN-CASH-16`).
3. **Given** el mismo turno, **When** se registra un movimiento `kind='retiro'` de $20.000,
   **Then** `expected` baja en $20.000 — egreso y retiro restan exactamente igual en la fórmula,
   solo se distinguen por su categoría reportable, no por su efecto aritmético (`RN-CASH-14`,
   `RN-CASH-16`).
4. **Given** cualquiera de los tres tipos, **When** se intenta registrar con `amount` en cero o
   negativo, **Then** la base de datos lo rechaza (`CheckConstraint`) — `amount` siempre debe ser
   estrictamente positivo, el signo lo aporta `kind`, nunca el monto (`RN-CASH-14`).
5. **Given** un turno ya cerrado, **When** se intenta registrar cualquier movimiento manual sobre
   él, **Then** el sistema rechaza con `409` "El turno está cerrado" (`RN-CASH-12`).

---

### User Story 6 - [Anomalía A-20/RN-CASH-13 — REQUISITO DE NEGOCIO NUEVO, confirmado en ronda 2] El arqueo parcial exige observación igual que el cierre real (Priority: P2)

Un cajero, a mitad de turno, hace un arqueo parcial (conteo físico sin cerrar el turno) para
verificar que el cajón cuadra hasta ese momento. Si la diferencia entre lo contado y lo esperado
es distinta de cero, el sistema **debe** exigir una nota explicando la diferencia — exactamente
la misma exigencia que ya aplica al cierre real (`RN-CASH-10`, User Story 4).

**Why this priority**: cierra una asimetría que el propio negocio, al ser consultado, calificó
como indeseada: no hay razón para exigir menos rigor a un arqueo parcial que a uno de cierre si
ambos reportan una diferencia real de efectivo.

**Comportamiento ACTUAL (el gap que este requisito cierra)**: hoy `POST
/cash/shifts/{id}/partial-count` calcula y persiste `difference` sin exigir ningún `note`, sin
importar cuán grande sea la diferencia — a diferencia de `close_shift`, que si aplica
`RN-CASH-10`. Este comportamiento actual queda documentado como el defecto a corregir, no como el
contrato válido.

**Independent Test**: se puede probar con `POST /cash/shifts/{id}/partial-count` provocando una
diferencia distinta de cero, una vez sin `note` y otra vez con `note`, verificando que solo la
segunda se acepta.

**Acceptance Scenarios (comportamiento DESEADO, ya confirmado por negocio — P14-bis)**:

1. **Given** un turno abierto con `expected=137.000`, **When** un cajero hace un arqueo parcial
   con `counted_amount=135.000` (diferencia de -$2.000) sin `note`, **Then** el sistema **debe**
   rechazar la operación con el mismo criterio que `RN-CASH-10` — "la observación es obligatoria
   cuando el arqueo no cuadra" (`RN-CASH-13`).
2. **Given** el mismo escenario, **When** el cajero incluye una `note` no vacía explicando la
   diferencia, **Then** el arqueo parcial se registra normalmente, igual que hoy (`RN-CASH-13`).
3. **Given** un arqueo parcial con diferencia exactamente cero, **Then** `note` sigue siendo
   opcional — el requisito nuevo solo aplica cuando hay diferencia real, igual que en el cierre
   (`RN-CASH-13`, `RN-CASH-10`).
4. **Given** un turno ya cerrado, **When** se intenta un arqueo parcial sobre él, **Then** el
   sistema sigue rechazando con `409` "El turno está cerrado" — esta regla no cambia
   (`RN-CASH-13`, comportamiento ya vigente).

---

### User Story 7 - [Anomalía A-20/RN-CASH-17 — REQUISITO DE NEGOCIO NUEVO, más estricto que "congelar", surgido en P14] El cierre de un turno es un hecho inmutable; las anulaciones posteriores se ven en una vista de ajustes separada (Priority: P1)

Un administrador consulta el histórico de turnos cerrados (para gestoría, auditoría o control).
Lo que ve para un turno ya cerrado —monto esperado, diferencia, conteo— **debe** ser exactamente
lo que se calculó y aceptó en el momento del cierre, para siempre, sin importar qué pase después
con las ventas que participaron en ese cálculo. Si una venta de un turno ya cerrado se anula
después, ese efecto **debe** verse en una vista de ajustes separada vinculada al turno original —
nunca sobrescribiendo ni recalculando el cierre original.

**Why this priority**: la gestoría usa el conteo de efectivo reportado por el arqueo
(`registro-de-anomalias.md`, R7) como registro contable de referencia; un histórico que cambia
retroactivamente delante de una auditoría —"el cierre del martes ahora dice otra cosa que el
martes"— es peor que una diferencia real sin explicar, porque rompe la confianza en el propio
registro. Por eso el negocio pidió, sin que se le preguntara directamente, un estándar más
estricto que "congelar en la próxima lectura": el dato ya persistido no se toca nunca.

**Comportamiento ACTUAL (el gap que este requisito cierra)**: hoy ni `list_shifts` (histórico) ni
`shift_report` (reporte de un turno) leen un valor guardado de `expected`/`difference` — ambos
llaman a `service.reconcile(db, shift)` en cada consulta, que vuelve a sumar `Sale`/`Payment` con
los datos **actuales** de la base. Si una venta que participó en el cálculo original se anula
después del cierre, la siguiente consulta del histórico de ese turno muestra un
`expected`/`difference` distintos a los que el cajero vio y aceptó al cerrar — sin ningún rastro
de que el número cambió, ni de por qué.

**Independent Test**: se puede probar cerrando un turno, registrando el `expected`/`difference`
que devuelve el cierre, anulando después una venta que participó en ese cálculo, y comparando el
`expected`/`difference` que devuelve una consulta posterior del histórico contra los valores
originales.

**Acceptance Scenarios (comportamiento DESEADO, ya confirmado por negocio — P14, más estricto que
"congelar")**:

1. **Given** un turno que se cierra con `expected=137.000`, `difference=-2.000` (con su
   `close_note` correspondiente), **Then** esos dos valores **deben** quedar persistidos como
   parte del hecho del cierre, no solo derivables — no un simple "recalcular pero cachear", sino
   un dato que el sistema nunca vuelve a tocar (`RN-CASH-17`).
2. **Given** ese turno ya cerrado, **When** después se anula una venta en efectivo que formaba
   parte del cálculo original de `ventas_efectivo`, **Then** una consulta posterior del histórico
   de ese turno **debe** seguir mostrando `expected=137.000`, `difference=-2.000` — exactamente
   los mismos valores que el cajero vio al cerrar, sin recalcular (`RN-CASH-17`).
3. **Given** la misma anulación posterior, **When** se necesita reflejar su efecto real sobre el
   efectivo del negocio, **Then** ese efecto **debe** aparecer como un registro de ajuste
   distinto, vinculado al turno original por referencia, visible en una vista separada de
   ajustes — nunca como una modificación del cierre original ni de sus campos persistidos
   (`RN-CASH-17`).
4. **Given** un turno que **todavía está abierto** (no cerrado), **When** se consulta su arqueo,
   **Then** el sistema **sigue** recalculando en tiempo real contra datos actuales, exactamente
   como documenta `RN-CASH-02` a `RN-CASH-06` (User Story 2) — el snapshot inmutable aplica
   únicamente a partir del momento del cierre, nunca a un turno en curso.

**Nota de alcance — vista de ajustes**: esta spec fija que la vista de ajustes debe existir y que
el cierre original nunca se sobrescribe; el diseño detallado de esa vista (qué campos muestra,
cómo se navega desde el turno original) es una decisión de diseño de la fase de planificación
(`/speckit-plan`), no de esta spec.

---

### User Story 8 - [Anomalía A-17, `ACCIDENTAL`] Cierre de turno y movimientos manuales sin bloqueo de fila (Priority: P2)

Dos operaciones concurrentes sobre el mismo turno —dos clics de cierre casi simultáneos, o un
movimiento manual registrado justo cuando otro cajero está cerrando el turno— pueden leer el
mismo estado "a medio comitear" porque ninguna de las dos bloquea la fila del turno antes de
leer y decidir, a diferencia de otros módulos del sistema (`inventory/stock.py`,
`invoices/service.py`, `table_sessions.close_session`) que sí usan bloqueo pesimista consistente
(`SELECT ... FOR UPDATE`, ver spec 005 User Story 2 para el patrón ya establecido con
`lock_items`).

**Why this priority**: sin este bloqueo, un doble clic accidental en "cerrar turno" puede insertar
denominaciones duplicadas (R7) y un movimiento manual registrado justo al filo del cierre puede
quedar fuera del cálculo de diferencia que el cajero ya vio y aceptó (R22) — ambos casos rompen
la integridad del mismo dato que la User Story 7 promete congelar.

**Independent Test**: se puede probar disparando dos requests concurrentes de `POST
/cash/shifts/{id}/close` con las mismas denominaciones sobre el mismo turno, y verificando si se
insertan denominaciones duplicadas; y disparando un `POST /cash/shifts/{id}/movements` en
paralelo con un `POST /cash/shifts/{id}/close` sobre el mismo turno.

**Acceptance Scenarios (comportamiento DESEADO — corrección de un defecto accidental, sin
decisión de negocio pendiente)**:

1. **Given** un turno abierto, **When** llegan dos requests de cierre casi simultáneos con las
   mismas denominaciones, **Then** el sistema **debe** bloquear la fila del turno
   (`SELECT ... FOR UPDATE`) antes de leer su `status`, de modo que el segundo request encuentre
   el turno ya `closed` y se rechace con `409` — nunca debe insertar las denominaciones dos veces.
2. **Given** un turno abierto, **When** un movimiento manual y un cierre de turno llegan casi al
   mismo tiempo, **Then** el sistema **debe** serializarlos con el mismo bloqueo, de forma que el
   movimiento manual quede incluido en el cálculo de `expected`/`difference` que ve el cierre, o
   sea rechazado limpiamente por turno ya cerrado — nunca debe quedar registrado pero excluido en
   silencio del arqueo que el cajero ya aceptó.

**Anomalía A-17 (porción caja) — clasificación `ACCIDENTAL`, corregir en fase de modernización**:
a diferencia de las User Stories 6 y 7 (requisitos de negocio), esta no requiere ninguna decisión
de negocio — es una inconsistencia técnica verificable contra el propio patrón que el resto del
código ya sigue. **Tratamiento acordado**: agregar `with_for_update()` en `close_shift` y
`add_movement`, siguiendo exactamente el mismo patrón ya usado en `lock_items`
(`app/api/v1/inventory/stock.py`, ver spec 005).

---

### User Story 9 - [Anomalía A-40/RN-CASH-15, `DUDOSA`/`PENDIENTE`] Alias deprecado `cash_sales`, idéntico a `ventas_efectivo` (Priority: P3)

Un consumidor legado del reporte de arqueo lee el campo `cash_sales`, mantenido únicamente por
compatibilidad con versiones anteriores del frontend. Su valor es exactamente igual a
`ventas_efectivo` (ya neto del cambio entregado) — nunca el bruto de pagos en efectivo, pese a que
su nombre podría sugerir eso.

**Why this priority**: es deuda técnica documentada, de bajo impacto funcional (ambos campos
siempre coinciden), pero con una pregunta abierta que condiciona si puede retirarse sin romper
nada.

**Independent Test**: se puede probar comparando `cash_sales` y `ventas_efectivo` en la respuesta
de `service.reconcile` sobre el mismo turno — deben ser siempre el mismo valor.

**Acceptance Scenarios**:

1. **Given** un turno con `by_type["cash"]=120.000` y `change_total=15.000`, **Then** tanto
   `ventas_efectivo` como `cash_sales` valen `105.000` — nunca `120.000`, pese a que el nombre
   `cash_sales` podría sugerir el bruto (`RN-CASH-15`).
2. **Given** cualquier turno, **Then** `cash_sales` y `ventas_efectivo` son siempre exactamente
   el mismo valor — no hay ningún escenario donde diverjan, porque el primero se calcula como
   alias directo del segundo (`RN-CASH-15`).

**Anomalía A-40 — clasificación `PENDIENTE`, tratamiento cerrado (documentar sin especificar
retiro)**: hay evidencia de código y de historia de negocio
(`memoria-historica.md` #17, 2026-07-18, commit `927a4606`, Deimer Hernandez) de que es deprecado
a propósito por compatibilidad, pero falta confirmar si el frontend ya migró a `ventas_efectivo`.
Esta spec documenta el alias tal cual existe hoy, **sin** fijar su retiro como parte del contrato
— retirarlo antes de confirmar la migración rompería cualquier consumidor que aún lo lea.
**Pregunta que la desbloquearía**: ¿sigue el frontend leyendo `cash_sales` hoy, o ya migró a
`ventas_efectivo` y el alias se puede retirar? Responsable: por identificar.

---

### Edge Cases

- **Cierre con denominaciones Y `counted_amount` en el mismo request**: las denominaciones ganan
  siempre; el `counted_amount` recibido se ignora por completo, no se usa como verificación
  cruzada ni genera advertencia (`RN-CASH-09`).
- **Arqueo parcial sobre un turno que se cierra segundos después**: bajo el comportamiento
  deseado de la User Story 8 (lock consistente), ambas operaciones deben serializarse; sin el
  lock (comportamiento accidental actual), el arqueo parcial puede seguir aceptándose incluso
  después de que el turno ya fue marcado `closed` en otra transacción concurrente.
- **Movimiento manual con `amount` exactamente cero**: rechazado por el `CheckConstraint` de base
  de datos — no es un caso silencioso, es un rechazo explícito a nivel de esquema (`RN-CASH-14`).
- **Un método de pago se desactiva a mitad de turno, después de haber tenido ventas**: el
  desglose (`RN-CASH-07`) solo garantiza incluir en cero los métodos **activos** al momento de la
  consulta; un método desactivado con ventas reales en el turno sigue apareciendo porque tiene
  filas en `vendido` por sus pagos, no por el bucle de "activos en cero" — este caso no está
  cubierto explícitamente por ninguna regla `RN-CASH` dedicada.
- **Consulta del histórico de un turno que sigue abierto** (`status='open'`) desde el endpoint de
  histórico (`GET /cash/shifts`, que filtra por defecto `status='closed'`): fuera del
  comportamiento por defecto documentado aquí; si se filtra explícitamente por `status='open'`,
  esos turnos siguen recalculando en tiempo real (User Story 7, escenario 4), nunca aplican el
  snapshot inmutable.
- **Anulación de una venta que afecta a un turno cerrado hace varios meses**: la User Story 7
  aplica sin importar cuánto tiempo haya pasado desde el cierre — no hay ventana de prescripción
  documentada para cuándo deja de aplicar la inmutabilidad.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: El sistema NO DEBE permitir que coexistan dos turnos en estado `open` para la misma
  caja registradora; un segundo intento de apertura DEBE rechazarse con `409` (`RN-CASH-01`).
- **FR-002**: El efectivo esperado del cajón DEBE derivarse en cada cálculo de los pagos reales
  (`Payment`) de ventas (`Sale`) con `status='paid'` asociadas al turno — nunca de un registro
  propio de "ventas" guardado en `cash_movements` (`RN-CASH-02`).
- **FR-003**: Únicamente los pagos de tipo efectivo (`PaymentMethod.type='cash'`) DEBEN sumar o
  restar al efectivo esperado del cajón; tarjeta y transferencia DEBEN reportarse en el desglose
  sin afectar ese monto (`RN-CASH-03`).
- **FR-004**: El efectivo esperado por ventas DEBE calcularse neto del cambio entregado (pagos en
  efectivo brutos menos la suma de `change_given` de las ventas del turno) (`RN-CASH-04`).
- **FR-005**: El efectivo esperado total al cierre DEBE seguir la fórmula:
  `fondo_inicial + ventas_efectivo_neto + ingresos_manuales - egresos_manuales -
  retiros_manuales` (`RN-CASH-05`).
- **FR-006**: La diferencia del arqueo (`counted_amount - expected`) DEBE calcularse únicamente
  cuando el turno tiene un `counted_amount` registrado; si no lo tiene, la diferencia DEBE quedar
  `None`, nunca asumirse `0` (`RN-CASH-06`).
- **FR-007**: El desglose de ventas por método de pago DEBE incluir todos los métodos de pago
  activos del negocio, aunque no hayan tenido ninguna venta en el turno, mostrando `total=0,
  count=0` para esos casos (`RN-CASH-07`).
- **FR-008**: En el desglose de ventas, el método de tipo efectivo DEBE aparecer siempre primero;
  el resto DEBE ordenarse alfabéticamente por nombre, sin distinguir mayúsculas de minúsculas
  (`RN-CASH-08`).
- **FR-009 [Anomalía A-20/RN-CASH-09, `DUDOSA`, mitigada operativamente — P14]**: al cerrar un
  turno, el monto contado DEBE tomarse de la suma de denominaciones si se envían; si no se
  envían denominaciones, DEBE tomarse del `counted_amount` recibido directamente; si no se envía
  ninguno de los dos, el conteo DEBE quedar `None`. El backend por sí solo no exige que se envíe
  al menos uno de los dos — el negocio confirmó que la pantalla de cierre siempre exige el
  conteo antes de permitir cerrar, así que esta spec no agrega una validación adicional a nivel
  de API (`RN-CASH-09`).
- **FR-010**: Al cerrar un turno, si la diferencia entre lo contado y lo esperado es distinta de
  cero, una observación (`close_note`) no vacía DEBE ser obligatoria; si la diferencia es cero,
  DEBE ser opcional (`RN-CASH-10`).
- **FR-011**: El sistema NO DEBE permitir cerrar un turno que ya está en estado `closed`; el
  intento DEBE rechazarse con `409`, sin modificar ningún dato del cierre original (`RN-CASH-11`).
- **FR-012**: Los movimientos manuales de efectivo (ingreso, egreso, retiro) DEBEN poder
  registrarse únicamente mientras el turno está en estado `open`; sobre un turno cerrado DEBEN
  rechazarse con `409` (`RN-CASH-12`).
- **FR-013 [Anomalía A-20/RN-CASH-13 — REQUISITO DE NEGOCIO NUEVO, confirmado en ronda 2, P14-bis
  — NO es el comportamiento actual]**: al registrar un arqueo parcial (sin cerrar el turno), si la
  diferencia entre lo contado y lo esperado es distinta de cero, una observación (`note`) no
  vacía DEBE ser obligatoria — exactamente la misma exigencia que `FR-010` aplica al cierre real.
  El comportamiento actual (que no exige nota en el arqueo parcial bajo ninguna circunstancia)
  queda documentado como el defecto que este requisito corrige, no como una alternativa válida
  (`RN-CASH-13`).
- **FR-014**: Solo DEBEN existir tres tipos de movimiento manual de efectivo — `ingreso` (suma),
  `egreso` (resta), `retiro` (resta) —, y el monto (`amount`) de cualquiera de los tres DEBE ser
  siempre estrictamente positivo; el signo de la operación lo determina el tipo (`kind`), nunca
  el signo de `amount` (`RN-CASH-14`).
- **FR-015 [Anomalía A-40/RN-CASH-15, `DUDOSA`/`PENDIENTE` — documentado sin especificar su
  retiro]**: el campo deprecado `cash_sales` DEBE seguir siendo exactamente igual a
  `ventas_efectivo` (ya neto del cambio entregado) mientras exista, mantenido únicamente por
  compatibilidad con consumidores que aún lo lean. Esta spec no fija su retiro como parte del
  contrato hasta confirmar que ningún consumidor real depende de él (`RN-CASH-15`).
- **FR-016**: Egresos y retiros DEBEN restar de forma idéntica en la fórmula del efectivo
  esperado (`FR-005`); solo DEBEN distinguirse por su categoría reportable, nunca por su efecto
  aritmético (`RN-CASH-16`).
- **FR-017 [Anomalía A-20/RN-CASH-17 — REQUISITO DE NEGOCIO NUEVO, más estricto que "congelar",
  surgido en la entrevista P14 — NO es el comportamiento actual]**: al cerrar un turno, el
  `expected` y el `difference` calculados en ese momento DEBEN persistirse como parte
  inseparable del hecho del cierre. Cualquier consulta posterior del histórico de un turno ya
  cerrado (listado, reporte) DEBE devolver esos valores persistidos, sin volver a ejecutar el
  cálculo contra datos actuales. Si una venta que participó en el cálculo original se anula
  después del cierre, ese efecto DEBE registrarse como un ajuste separado, vinculado al turno
  original por referencia, visible en una vista de ajustes propia — el cierre original NUNCA
  DEBE modificarse ni sobrescribirse. Esta regla aplica únicamente a turnos ya cerrados; un turno
  todavía abierto sigue recalculando en tiempo real según `FR-002` a `FR-006` (`RN-CASH-17`). El
  comportamiento actual (recalcular siempre contra datos vivos, incluso para turnos cerrados)
  queda documentado como el defecto que este requisito corrige.
- **FR-018 [Anomalía A-17, `ACCIDENTAL` — corregir en fase de modernización, sin decisión de
  negocio pendiente]**: `close_shift` y `add_movement` DEBEN adquirir un bloqueo de fila
  (`SELECT ... FOR UPDATE`) sobre el turno antes de leer su estado y decidir, consistente con el
  patrón ya establecido en el resto del sistema (`lock_items` en inventario, `invoices/service.py`,
  `table_sessions.close_session`) — para que un doble clic en cerrar turno no duplique
  denominaciones (R7) y un movimiento manual justo al filo del cierre no quede excluido en
  silencio de la diferencia reportada (R22).

### Key Entities *(include if feature involves data)*

- **CashRegister**: caja registradora física del negocio. Tiene un nombre único; puede tener a lo
  sumo un `CashShift` en estado `open` a la vez (`RN-CASH-01`).
- **CashShift**: turno de caja. Atributos relevantes: `cash_register_id`, `user_id`/`user_name`
  (quien abrió), `opening_amount`, `status` (`open`/`closed`), `counted_amount`, `close_note`,
  `opened_at`, `closed_at`. A partir de `FR-017`, un turno cerrado también DEBE conservar el
  `expected`/`difference` calculados en el momento del cierre como parte de su propio registro
  (o de un registro vinculado equivalente), no solo derivarlos en cada lectura.
- **CashMovement**: movimiento manual de efectivo dentro de un turno abierto. Atributos:
  `cash_shift_id`, `kind` (`ingreso`/`egreso`/`retiro`), `amount` (siempre positivo), `category`,
  `description`, `occurred_at`, `user_id`/`user_name` (`RN-CASH-12`, `RN-CASH-14`).
- **CashCountDenomination**: detalle de billete/moneda y cantidad reportado al cerrar un turno,
  cuando el cierre se hace por denominaciones (`RN-CASH-09`).
- **CashPartialCount**: registro de un arqueo parcial (sin cerrar el turno): `cash_shift_id`,
  `counted_amount`, `expected_amount`, `difference`, `note`, `user_id`/`user_name`. A partir de
  `FR-013`, `note` DEBE ser obligatoria cuando `difference != 0`.
- **Ajuste de caja** *(entidad nueva, requerida por `FR-017`, sin modelo de datos existente hoy)*:
  registro que vincula un turno ya cerrado con el efecto de un evento posterior (p. ej. una venta
  anulada) que habría cambiado su `expected`/`difference` si se recalculara. Vive fuera del
  registro del cierre original — nunca lo modifica.
- **Sale / Payment / PaymentMethod** *(fuera de alcance de generación — spec 011, consumidos aquí
  solo como fuente de lectura)*: el arqueo lee `Sale.cash_shift_id`, `Sale.status`,
  `Sale.change_given` y los `Payment` asociados, agrupados por `PaymentMethod.type`
  (`RN-CASH-02` a `RN-CASH-04`).

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: El 100% de las reglas `RN-CASH-01` a `RN-CASH-17` puede verificarse ejecutando los
  pasos descritos en esta spec contra un `pos-backend` en ejecución, sin necesitar leer
  `cash/service.py` ni `cash/router.py` para entender el comportamiento esperado (vigente o
  deseado, según corresponda a cada regla).
- **SC-002 [Gap de caracterización, prioritario]**: no existe hoy ningún characterization test
  dedicado a `cash/service.py`/`cash/router.py` entre los 12 scripts de test existentes en el
  repositorio. Cerrar este gap es prioritario, y en particular **antes** de implementar `FR-013`
  y `FR-017` (los dos requisitos nuevos): sin un test que capture primero el comportamiento
  actual de `partial-count` y del histórico recalculado, no hay forma de verificar que la
  corrección realmente cambió el comportamiento y no introdujo una regresión adicional.
- **SC-003**: La anomalía A-20/RN-CASH-13 queda cerrada como requisito de negocio confirmado: el
  100% de los arqueos parciales con diferencia distinta de cero exigen una observación no vacía,
  igual que el cierre real.
- **SC-004**: La anomalía A-20/RN-CASH-17 queda cerrada como requisito de negocio confirmado, más
  estricto que "congelar": el 100% de los turnos cerrados consultados desde el histórico
  muestran el `expected`/`difference` exactos del momento del cierre, sin importar cuántas
  ventas relacionadas se hayan anulado después; el efecto de esas anulaciones es visible
  exclusivamente en una vista de ajustes separada, nunca sobrescribiendo el cierre original.
- **SC-005**: La anomalía A-17 (porción caja) queda cerrada: ni un doble clic en cerrar turno
  produce denominaciones duplicadas, ni un movimiento manual concurrente con un cierre queda
  excluido en silencio del cálculo de diferencia — verificable con un test de concurrencia
  dedicado sobre `close_shift` y `add_movement`.
- **SC-006**: La anomalía A-40 (alias `cash_sales`) queda documentada con su equivalencia exacta a
  `ventas_efectivo` y su pregunta de negocio pendiente registrada; su retiro no ocurre como parte
  de esta spec, evitando romper cualquier consumidor del frontend que aún no haya migrado.

## Out of Scope

- **Cómo se generan las ventas que el arqueo agrega** (creación de `Sale`/`Payment`, los cuatro
  caminos de cobro) — cubierto por la spec 011, aún no escrita en este reconocimiento. Esta spec
  solo documenta cómo el arqueo **lee** esos datos ya existentes.
- **El cierre de cuenta de mesa en sí** (`table_sessions.close_session`) — cubierto por la spec
  010, aún no escrita en este reconocimiento. Se cita aquí únicamente como referencia del patrón
  de bloqueo de fila que `FR-018` debe replicar.
- **El diseño detallado de la vista de ajustes** requerida por `FR-017` (campos exactos, UI,
  navegación desde el turno original) — esta spec fija que debe existir y que nunca debe
  modificar el cierre original; el diseño concreto es tarea de `/speckit-plan`.
- **Confirmar si el frontend ya migró de `cash_sales` a `ventas_efectivo`** (A-40) — pregunta de
  negocio abierta, no resuelta por esta spec; se documenta la equivalencia y se deja la decisión
  de retiro pendiente.
- **Administración de cajas registradoras** (`CashRegister`: creación, listado) — se menciona en
  User Story 1 solo como precondición de apertura de turno; su CRUD administrativo no tiene
  reglas `RN-CASH` propias más allá de la unicidad de nombre.

## Assumptions

- **Esta es una spec de ingeniería inversa, no de una feature nueva, con dos excepciones
  explícitas**: a diferencia del resto de las guías de este template ("evitar detalles de
  implementación"), aquí los nombres de campo, endpoints, mensajes de error y fórmulas citados
  **son** el contrato observable que se está documentando. Las excepciones son `RN-CASH-13`
  (`FR-013`) y `RN-CASH-17` (`FR-017`): ahí el contrato fijado es el **deseado y ya confirmado
  por negocio**, no el actual — cualquier characterization test que se escriba para estas dos
  reglas debe capturar primero el comportamiento actual como regresión conocida (ver `SC-002`),
  y luego verificar el comportamiento nuevo tras la corrección.
- **`RN-CASH-09` (A-20, parte 1) se documenta tal cual, sin agregar validación de backend**: el
  negocio confirmó que la mitigación real vive en la pantalla de cierre (siempre exige el conteo
  antes de habilitar el botón), no en el contrato de la API. Si en el futuro aparece un cliente
  distinto al frontend actual que llame directamente a `close_shift`, esta mitigación deja de
  cubrir el riesgo y la pregunta debería reabrirse.
- **`A-17` (porción caja) no requiere decisión de negocio**: a diferencia de `RN-CASH-13` y
  `RN-CASH-17`, es una inconsistencia técnica verificable por contraste directo con el patrón que
  el resto del código ya sigue (`lock_items`, spec 005) — se corrige en fase de modernización sin
  esperar respuesta de negocio.
- **`A-40` permanece `PENDIENTE`, sin fecha de retiro**: el alias `cash_sales` se documenta y se
  mantiene funcionando idéntico a `ventas_efectivo` hasta que se confirme que el frontend ya
  migró; esta spec no asume una respuesta a esa pregunta.
- **Los valores numéricos y de ejemplo citados en los escenarios** (`opening_amount=50.000`,
  `expected=137.000`, denominaciones, nombres de métodos de pago) son ilustrativos, tomados de
  `reglas-de-negocio.md` y `registro-de-anomalias.md` — no representan necesariamente datos reales
  de producción.
- **El gap de caracterización (`SC-002`) es una brecha documentada, no una tarea de esta spec**:
  esta spec no crea el characterization test faltante — señala su ausencia como prioridad,
  especialmente antes de tocar código para implementar `FR-013`/`FR-017`, donde un test que
  capture primero el comportamiento actual es la única forma de verificar que la corrección
  cambió lo que debía cambiar sin romper el resto (`RN-CASH-01` a `12`, `14` a `16`).
