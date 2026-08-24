# Feature Specification: Corrección global de fechas, horas y zonas horarias

**Feature Branch**: `030-correccion-fechas-zona-horaria`

**Created**: 2026-08-24

**Status**: Draft

**Input**: User description: "Corrección global de fechas, horas y zonas horarias — el módulo de
Ventas muestra una venta con fecha/hora `24/08/2026 12:49` mientras la hora real del negocio en
Bogotá era `07:53`. Se pide auditar y corregir de forma centralizada y reutilizable, no aislada por
pantalla, cómo se generan, almacenan, transportan y muestran las fechas/horas en todo el sistema
(ventas, órdenes, pagos, caja, inventario, mesas, facturas, compras, auditoría), definir una
estrategia única, evaluar zona horaria configurable por tenant, y evitar cambios retroactivos sobre
datos históricos."

**Naturaleza de esta spec**: **spec de corrección de comportamiento existente**, no de característica
nueva (Principio II de la [Constitución](../../.specify/memory/constitution.md)). Corrige un defecto
de visualización/interpretación de zona horaria que hoy afecta, con el mismo patrón, a todas las
entidades del sistema que registran un instante absoluto (ver Key Entities). No modifica el motor de
evaluación de vigencia de promociones (`active_discount_promotions`/`local_now`/`best_line_discount`
— spec [012](../012-motor-de-evaluacion-de-promociones-y-combos/spec.md), anomalía A-07,
**protegida, no se toca**), ni duplica las correcciones ya definidas en las specs
[022-correccion-zona-horaria-menu-carrito](../022-correccion-zona-horaria-menu-carrito/spec.md)
(A-08, menú/carrito QR) y
[023-correccion-zona-horaria-pos-staff](../023-correccion-zona-horaria-pos-staff/spec.md) (A-09, POS
de staff) — ambas sobre la *vigencia de promociones mostrada antes de cobrar*, un problema distinto
al de esta spec (*cómo se muestra la fecha/hora ya registrada de un hecho consumado*, como una venta,
un pago o un movimiento de inventario).

**Autorización de negocio (Principio II de la Constitución)**: el defecto reportado en Ventas
(`24/08/2026 12:49` mostrado vs. `07:53` real) fue verificado contra el código en ejecución
(`pos-backend`, `pos-heladeria`) el 2026-08-24 y se confirma que el mismo patrón afecta a todas las
entidades con timestamps de instante absoluto listadas en Key Entities (ver `research.md`/hallazgos
de esta spec para el detalle archivo:línea). El propietario del repositorio (deimerhdz21@gmail.com),
actuando como negocio, autoriza el 2026-08-24 corregir este comportamiento de forma centralizada y
reutilizable, y reabrir el tratamiento de la anomalía A-46 (`registro-de-anomalias.md`, "la zona
horaria de evaluación es un único valor global de la instancia, no por tenant") para introducir la
columna de zona horaria por tenant que su propio "Tratamiento acordado" preveía como paso siguiente.
La implementación de esta spec **debe** registrar en `registro-de-anomalias.md` una nueva entrada de
anomalía (siguiente disponible tras A-49) para el defecto de Ventas y las demás entidades, y actualizar
la entrada A-46 con esta decisión (ver `FR-011`).

**Nota de alcance importante (verificada leyendo el código real, no solo esta spec)**: la
investigación de esta spec confirma que **no hace falta migrar datos históricos**. Todas las columnas
de fecha/hora de instante absoluto del backend son `DateTime` sin zona horaria (`TIMESTAMP WITHOUT
TIME ZONE`), pobladas de forma consistente con `server_default=func.now()` de PostgreSQL o con
`datetime.now(timezone.utc)` desde la aplicación — es decir, **ya almacenan UTC de forma consistente**,
solo que sin marca de zona. El defecto está en que ningún punto de la cadena (schema de respuesta de
la API → recepción en Angular → `DatePipe`/`toLocaleString`) convierte ese UTC a la hora del negocio
antes de mostrarlo: el frontend interpreta la cadena ISO sin offset como si ya fuera hora local del
navegador. Es la corrección inversa y complementaria a A-08 (que trataba UTC naive como si ya fuera
hora local *al evaluar*, no al mostrar). La única excepción documentada es `Promotion.starts_at`/
`ends_at`, que representan una hora de pared local recurrente, no un instante — y por eso quedan
fuera de esta spec (ver `FR-009`, Out of Scope).

## Clarifications

### Session 2026-08-24

- Q: ¿Cómo se configura la zona horaria de un tenant (Historia 4)? → A: Como un campo configurable
  solo desde backend (columna en BD + migración), fijado vía el proceso de aprovisionamiento/soporte
  existente (script, acceso administrativo a BD o un endpoint interno) — sin pantalla nueva de
  autoservicio para el tenant en esta spec.
- Q: Si el valor de zona horaria de un tenant termina siendo inválido (no es un nombre IANA
  reconocido), ¿cuándo debe detectarse y qué debe pasar? → A: Se rechaza en el momento de escribirlo
  — el sistema valida contra la base de datos de zonas horarias IANA cada vez que el campo se
  establece (migración, script o endpoint interno); un valor inválido nunca llega a persistirse.

## User Scenarios & Testing *(mandatory)*

<!--
  Igual que las specs 007/012/022/023 de las que depende o con las que colinda, esta spec cita
  nombres de función, archivo y línea porque son el contrato observable que se está corrigiendo, no
  una fuga de detalles de implementación (ver Assumptions).
-->

### User Story 1 - Una venta muestra la hora real en que ocurrió (Priority: P1) — defecto reportado

Un cajero o administrador consulta el listado de Ventas o el detalle de una venta. Hoy la hora
mostrada (`sold_at`, columna `DateTime` sin zona poblada por `func.now()` de PostgreSQL, mostrada sin
conversión por `sales-page.component.ts:108,155` vía `DatePipe`) es la hora UTC cruda del servidor,
no la hora de Bogotá en que realmente se hizo la venta — una diferencia de 5 horas con
`TENANT_TIMEZONE=America/Bogota`.

**Why this priority**: es el defecto reportado, el punto de mayor exposición — Ventas es lo primero
que un cajero, administrador o auditor revisa para conciliar el día, y una hora incorrecta ahí genera
desconfianza inmediata en todo el sistema de reportes.

**Independent Test**: se puede probar de forma aislada creando una venta con el reloj fijado a un
instante conocido, consultando el listado/detalle de Ventas inmediatamente después, y verificando que
la hora mostrada coincide con la hora de Bogotá de ese instante, no con la hora UTC cruda.

**Acceptance Scenarios**:

1. **Given** una venta registrada a las 07:53 hora de Bogotá (12:53 UTC), **When** un usuario
   consulta el listado de Ventas o el modal de recibo, **Then** la hora mostrada es `07:53` —
   corrige el comportamiento actual, que hoy muestra `12:53`/`12:49`.
2. **Given** la misma venta, **When** se consulta vía la API directamente, **Then** el valor
   devuelto identifica sin ambigüedad que es UTC (no una cadena naive interpretable como local) —
   cualquier cliente que la reciba puede convertirla correctamente sin adivinar la zona horaria.

---

### User Story 2 - Todas las entidades con fecha/hora usan el mismo mecanismo, no una corrección por pantalla (Priority: P1)

El mismo patrón defectuoso (columna UTC sin zona, servida sin conversión, mostrada tal cual) se
repite en Órdenes (`CustomerOrder.created_at`), Pagos (`Payment.paid_at`), Caja
(`CashShift.opened_at/closed_at`, `CashMovement.occurred_at`, `CashPartialCount.counted_at`),
Inventario (`InventoryMovement.moved_at`), Mesas (`TableSession.opened_at/closed_at`,
`SessionParticipant.joined_at`), Facturas (`Invoice.issued_at`), Compras (`Purchase.purchased_at`) y
Auditoría (`AuditLog.at`, y `created_at`/`updated_at` de `TimestampMixin` en la mayoría de entidades).
El negocio necesita que la corrección se aplique una sola vez, de forma centralizada, y que las
demás pantallas la hereden — no que cada una se corrija de forma aislada con su propia lógica.

**Why this priority**: mismo peso que Historia 1 — sin un mecanismo único y reutilizable, cada
corrección aislada por pantalla reintroduce el mismo riesgo de inconsistencia que causó el defecto
original, y el negocio ya pidió explícitamente evitarlo.

**Independent Test**: se puede probar de forma aislada verificando, para cada entidad listada, que
su fecha se muestra convertida a la hora del negocio usando el mismo mecanismo de conversión (una
sola función/servicio central en backend y uno en frontend, no una por módulo).

**Acceptance Scenarios**:

1. **Given** un registro de cualquiera de las entidades listadas creado en un instante conocido,
   **When** se consulta desde su pantalla correspondiente, **Then** la hora mostrada coincide con la
   hora de Bogotá de ese instante.
2. **Given** el código fuente después de esta corrección, **When** se revisa cómo cada entidad
   convierte su fecha para mostrarla, **Then** todas usan el mismo mecanismo central de conversión
   (no una implementación de conversión distinta por módulo o componente).

---

### User Story 3 - Los filtros de fecha respetan el día del negocio, no el día UTC (Priority: P1)

Un administrador filtra Ventas o Reportes por "Hoy", "Ayer", o un rango "Desde/Hasta". Hoy el backend
compara `Sale.sold_at` contra medianoche UTC (`sales/service.py:206-209`,
`reports/service.py:21-25`), mientras el frontend de Reportes calcula "Hoy" en hora local del
navegador (`reports.service.ts:251-268`) — dos criterios de medianoche distintos que no coinciden con
la medianoche de Bogotá salvo coincidencia. Una venta hecha a las 23:30 o a las 00:15 hora de Bogotá
puede aparecer asociada al día equivocado.

**Why this priority**: mismo peso que las anteriores — es el escenario donde el defecto tiene
consecuencia operativa directa (cierre de caja, conciliación diaria, reportes fiscales), no solo
visual.

**Independent Test**: se puede probar de forma aislada creando registros a las 23:59, 00:00 y 00:01
hora de Bogotá y verificando que los filtros "Hoy"/"Ayer"/"Desde-Hasta" los asignan al día de Bogotá
correcto, no al día UTC.

**Acceptance Scenarios**:

1. **Given** una venta registrada a las 23:59 hora de Bogotá del día D, **When** se filtra por
   "Hoy" siendo D el día actual del negocio, **Then** la venta aparece incluida — corrige el
   comportamiento actual, que hoy puede excluirla por caer después de medianoche UTC del día
   anterior.
2. **Given** una venta registrada a las 00:01 hora de Bogotá del día D+1, **When** se filtra por
   "Hoy" siendo D el día actual del negocio, **Then** la venta NO aparece en el filtro de D —
   corrige el comportamiento actual, que hoy puede incluirla por caer antes de medianoche UTC.
3. **Given** un rango "Desde" el día D "Hasta" el día D, **When** se aplica el filtro, **Then**
   incluye exactamente las ventas entre las 00:00:00 y las 23:59:59 hora de Bogotá del día D.

---

### User Story 4 - El negocio puede configurar la zona horaria por tenant (Priority: P2) — reapertura de A-46

Hoy `TENANT_TIMEZONE` es un único valor global de la instancia (`config.py:20`, default
`America/Bogota`), y `Tenant` no tiene columna de zona horaria propia (`app/core/models.py:43-72`,
confirmado sin ese campo) — documentado como limitación conocida en A-46. El sistema es
multi-tenant (schema-per-tenant); el negocio necesita, como mínimo, que cada tenant pueda configurar
su propia zona horaria en vez de depender de un valor fijo de instancia, preparando el sistema para
operar negocios en ubicaciones distintas sin requerir un cambio de código por cada uno.

**Why this priority**: no bloquea la corrección del defecto reportado (que se soluciona con el
mecanismo central de Historias 1-3, usando `America/Bogota` como valor por defecto), pero es la
base para que la corrección sea sostenible a futuro sin repetir el mismo riesgo por cada tenant
nuevo con otra zona horaria.

**Independent Test**: se puede probar de forma aislada configurando dos tenants con zonas horarias
distintas — mediante el campo/columna correspondiente en base de datos, sin necesidad de una pantalla
de UI (ver Clarifications, esta spec no exige autoservicio de tenant) — y verificando que cada uno
muestra sus propias fechas convertidas a su propia zona horaria configurada, sin afectar al otro.

**Acceptance Scenarios**:

1. **Given** un tenant sin zona horaria configurada explícitamente, **When** se muestra cualquier
   fecha de ese tenant, **Then** se usa `America/Bogota` como valor por defecto — sin cambio de
   comportamiento para los tenants existentes.
2. **Given** un tenant con zona horaria configurada explícitamente a un valor distinto de
   `America/Bogota`, **When** se muestra cualquier fecha de ese tenant, **Then** se convierte usando
   esa zona horaria configurada, no la de otro tenant ni la del servidor.

---

### User Story 5 - El valor de fecha/hora que el usuario elige en un formulario no cambia al enviarse o recuperarse (Priority: P2)

Un usuario selecciona una fecha/hora en un formulario (filtro Desde/Hasta, ventana de vigencia de una
promoción). El negocio necesita la garantía de que ese valor, tal como el usuario lo seleccionó, no
cambia de forma involuntaria al enviarse al backend o al recuperarse después en pantalla.

**Why this priority**: sin esta garantía, corregir la visualización de fechas ya registradas
(Historias 1-3) podría introducir un defecto simétrico en la entrada de datos — el mismo tipo de
error, en la dirección contraria.

**Independent Test**: se puede probar de forma aislada seleccionando una fecha/hora en un formulario,
enviándola al backend, y verificando que al recuperarla y mostrarla de nuevo el valor es idéntico al
seleccionado originalmente.

**Acceptance Scenarios**:

1. **Given** un usuario selecciona el 24/08/2026 como filtro "Desde", **When** el filtro se aplica y
   se vuelve a mostrar en el formulario, **Then** sigue mostrando 24/08/2026 — no 23/08/2026 ni
   25/08/2026 por un corrimiento de zona horaria.

---

### User Story 6 - Ningún dato histórico ya registrado cambia de valor (Priority: P1) — Principio VII

El equipo de modernización/evolución necesita la certeza de que esta corrección, al tocar cómo se
generan y muestran fechas en prácticamente todo el sistema, no recalcula ni altera el valor
almacenado de ninguna venta, pago, factura o cualquier otro registro ya existente — solo cambia cómo
se interpreta y se presenta.

**Why this priority**: sin esta garantía, el alcance sistémico de esta corrección (todas las
entidades del sistema) sería el cambio de mayor riesgo posible sobre datos de producción — viola
directamente el Principio VII de la Constitución (compatibilidad e inmutabilidad de datos
históricos, en particular de facturas ya emitidas).

**Independent Test**: se puede probar de forma aislada comparando, para un conjunto de registros
existentes antes de esta corrección, su valor almacenado en base de datos antes y después de
desplegar el cambio — debe ser idéntico en todos los casos.

**Acceptance Scenarios**:

1. **Given** una venta, un pago o una factura ya registrados antes de esta corrección, **When** se
   inspecciona su valor almacenado en base de datos después de desplegar la corrección, **Then** el
   valor no cambió — solo cambió cómo se muestra.
2. **Given** una factura ya emitida, **When** se consulta después de esta corrección, **Then** su
   importe y su representación de fecha de emisión siguen siendo exactamente las mismas que antes
   (Principio VII de la Constitución).

---

### Edge Cases

- ¿Qué pasa en el instante exacto de medianoche (`23:59`, `00:00`, `00:01` hora de Bogotá)? Ver
  Historia 3 — debe asignarse al día de Bogotá correcto, no al día UTC.
- ¿Colombia usa horario de verano (DST)? No — `America/Bogota` no observa DST; esta corrección no
  necesita lógica de cambio de horario estacional, solo el offset fijo UTC-5.
- ¿Qué pasa con `Promotion.starts_at`/`ends_at`? Fuera de alcance — representan una hora de pared
  local recurrente (ventana de vigencia), no un instante absoluto; su tratamiento ya está definido
  por A-07 (protegido) y no cambia aquí (ver Out of Scope).
- ¿Qué pasa con los puntos donde hoy se asigna `datetime.now(timezone.utc)` (aware) directo sobre una
  columna sin zona (`cash/router.py:121`, `checkout.py:811/873/925`, `qr_context.py:85/179`,
  `table_sessions/service.py:177/652/739`)? Deben unificarse para usar el mismo mecanismo central de
  generación de "ahora" (ver `FR-002`/`FR-010`) — el valor final no cambia (ya es UTC), solo se
  elimina la inconsistencia de que cada punto construya su propio `datetime.now(...)`.
- ¿Qué pasa si un tenant no tiene zona horaria configurada? Ver Historia 4, Escenario 1 — usa
  `America/Bogota` por defecto, sin romper el comportamiento actual.
- ¿Qué pasa si se intenta establecer una zona horaria de tenant con un valor que no es un nombre IANA
  reconocido? Se rechaza en el momento de escribirlo (validación contra la base de datos IANA); el
  valor inválido nunca llega a persistirse (ver Clarifications, `FR-005`).
- ¿Qué pasa con los registros históricos ya almacenados con el patrón UTC-naive actual? Ver Historia
  6 — no se migran ni se alteran; el mecanismo de conversión los interpreta correctamente como UTC en
  el momento de leerlos y mostrarlos, sin tocar el dato almacenado.
- ¿Qué pasa si el servidor, el contenedor Docker o el navegador del usuario tienen configurada una
  zona horaria distinta a UTC o a la del negocio? La corrección no debe depender de esa
  configuración externa — el mecanismo central hace la conversión de forma explícita usando la zona
  horaria del tenant, sin asumir la del sistema operativo, contenedor o navegador.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001 [Corrige el defecto reportado, Historia 1]**: el sistema DEBE mostrar la fecha/hora de
  cualquier registro que represente un instante absoluto (venta, orden, pago, movimiento de caja,
  movimiento de inventario, apertura/cierre de mesa o sesión, factura, compra, entrada de auditoría)
  convertida a la zona horaria del negocio, no como la hora UTC cruda que hoy se muestra sin
  conversión.
- **FR-002 [Historia 2, mecanismo único]**: el sistema DEBE usar un único mecanismo de conversión
  UTC → hora del negocio, compartido por todas las entidades listadas en Key Entities, en lugar de
  una implementación de conversión distinta por módulo, componente o pantalla.
- **FR-003 [Historia 1/2, transporte inequívoco]**: toda fecha/hora que la API devuelva y represente
  un instante absoluto DEBE viajar en un formato que identifique sin ambigüedad que es UTC (o incluya
  su offset), de forma consistente entre todos los endpoints — eliminando la mezcla actual entre
  campos que sí llevan esa marca y campos que no.
- **FR-004 [Historia 3, filtros]**: los filtros "Desde", "Hasta", "Hoy", "Ayer", "Esta semana" y "Este
  mes" de Ventas y Reportes DEBEN calcular los límites del día usando la zona horaria del negocio, no
  medianoche UTC ni medianoche del navegador del usuario.
- **FR-005 [Historia 4, zona horaria por tenant]**: el sistema DEBE permitir configurar la zona
  horaria de cada tenant de forma independiente, usando `America/Bogota` como valor por defecto
  cuando no se configure explícitamente, sin requerir cambio de código para operar un tenant en una
  zona horaria distinta. Basta con un campo configurable desde backend (columna en base de datos,
  fijado vía el proceso de aprovisionamiento/soporte existente); esta spec NO exige una pantalla de
  autoservicio en la que el propio tenant edite su zona horaria (ver Clarifications). El sistema DEBE
  validar el valor contra la base de datos de zonas horarias IANA en el momento de establecerlo
  (migración, script o endpoint interno) y rechazar el valor si no es un nombre IANA reconocido — un
  valor inválido nunca debe llegar a persistirse (ver Clarifications).
- **FR-006 [Historia 5, formularios]**: el valor de fecha/hora que un usuario selecciona en un
  formulario (filtros, ventana de vigencia de una promoción, cualquier selector de fecha/hora) NO
  DEBE cambiar de forma involuntaria al enviarse al backend o al recuperarse después en pantalla.
- **FR-007 [Historia 6, Principio VII de la Constitución]**: esta corrección NO DEBE recalcular,
  revertir, migrar ni alterar el valor almacenado de ninguna venta, orden, pago, factura, movimiento
  de caja o inventario, ni ningún otro registro ya existente — el ajuste es exclusivamente de
  interpretación y presentación, dado que la investigación confirma que los timestamps de instante
  absoluto ya se almacenan consistentemente en UTC.
- **FR-008 [consistencia, cierre de brecha]**: los puntos de la aplicación donde hoy se construye la
  hora actual de forma independiente para asignarla a un registro (por ejemplo
  `datetime.now(timezone.utc)` repetido en múltiples archivos) DEBEN unificarse para usar el mismo
  mecanismo central de FR-002, de modo que ningún punto nuevo del sistema pueda reintroducir el mismo
  patrón defectuoso de forma aislada.
- **FR-009 [exclusión explícita, motor de promociones]**: la ventana de vigencia de una promoción
  (`Promotion.starts_at`/`ends_at`) DEBE seguir tratándose como hora de pared local del tenant, no
  como instante absoluto — el criterio de evaluación del motor de promociones (A-07, protegido) no
  cambia como resultado de esta spec.
- **FR-010 [Principio X de la Constitución, verificación]**: la implementación DEBE incluir, como
  mínimo, un test por cada una de las entidades Ventas, Órdenes, Pagos, Caja e Inventario que
  demuestre que un registro creado cerca de la medianoche en hora de Bogotá (23:59, 00:00, 00:01) se
  muestra y se filtra en el día correcto, más al menos un test que confirme que el valor almacenado
  de un registro histórico no cambia tras esta corrección.
- **FR-011 [Principio II de la Constitución, trazabilidad]**: la implementación DEBE registrar en
  `registro-de-anomalias.md` una nueva entrada de anomalía (siguiente disponible tras A-49) para el
  defecto de visualización descrito en esta spec, citando quién y cuándo autorizó la corrección, y
  DEBE actualizar la entrada A-46 documentando que la zona horaria pasa a ser configurable por
  tenant (Historia 4).

### Key Entities

- **Sale (Venta)**: entidad del defecto reportado; su fecha de instante absoluto es `sold_at`.
- **CustomerOrder (Orden)**: su fecha de creación (`created_at`) es un instante absoluto con el mismo
  patrón que Sale.
- **Payment (Pago)**: su fecha de pago (`paid_at`) es un instante absoluto con el mismo patrón.
- **CashShift (Turno de caja)**, **CashMovement (Movimiento de caja)**, **CashPartialCount (Arqueo
  parcial)**: sus fechas de apertura/cierre/movimiento/conteo (`opened_at`, `closed_at`,
  `occurred_at`, `counted_at`) son instantes absolutos con el mismo patrón; relevantes para el cierre
  diario de caja del negocio.
- **InventoryMovement (Movimiento de inventario)**: su fecha (`moved_at`) es un instante absoluto con
  el mismo patrón.
- **TableSession (Sesión de mesa)**, **SessionParticipant (Participante de sesión)**: sus fechas de
  apertura/cierre/ingreso (`opened_at`, `closed_at`, `joined_at`) son instantes absolutos con el mismo
  patrón; `expires_at` (TTL de sesión) queda fuera de alcance funcional visible al usuario pero usa el
  mismo mecanismo de "ahora" (FR-008).
- **Invoice (Factura)**: su fecha de emisión (`issued_at`) es un instante absoluto con el mismo
  patrón; sujeta además al Principio VII (inmutabilidad de facturas ya emitidas — ver FR-007).
- **Purchase (Compra)**: su fecha (`purchased_at`) es un instante absoluto con el mismo patrón.
- **AuditLog (Auditoría)**: su fecha (`at`) es un instante absoluto con el mismo patrón.
- **Registros con `TimestampMixin`** (`Tenant`, `User` y otras entidades base): sus campos
  `created_at`/`updated_at` son instantes absolutos con el mismo patrón.
- **Promotion (Promoción)**: su ventana de vigencia (`starts_at`/`ends_at`) es la única excepción
  documentada — hora de pared local recurrente, no instante absoluto; fuera de alcance (`FR-009`).
- **Tenant**: entidad que hoy no tiene zona horaria propia (limitación de A-46); esta spec exige que
  pueda configurarla de forma independiente (Historia 4, `FR-005`).

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: el 100% de las ventas consultadas desde cualquier pantalla muestra su hora idéntica a
  la hora real del negocio en que ocurrió, con un margen de error menor a un minuto.
- **SC-002**: las nueve entidades listadas en Key Entities con instante absoluto (Venta, Orden, Pago,
  Turno de caja, Movimiento de caja, Arqueo parcial, Movimiento de inventario, Sesión de mesa,
  Factura, Compra, Auditoría) muestran su fecha/hora usando el mismo mecanismo central de conversión,
  verificado por al menos un test por entidad.
- **SC-003**: el 100% de los registros creados en los tres minutos alrededor de la medianoche del
  negocio (23:59, 00:00, 00:01 hora de Bogotá) se filtra y se muestra asociado al día de Bogotá
  correcto, en Ventas y en Reportes.
- **SC-004**: el 100% de los registros existentes antes de esta corrección conserva su valor
  almacenado sin cambios, verificado por comparación directa contra base de datos antes y después del
  despliegue.
- **SC-005**: un tenant puede configurarse con una zona horaria distinta a `America/Bogota` sin
  requerir ningún cambio de código, y sus fechas se muestran correctamente en esa zona horaria sin
  afectar a otros tenants.
- **SC-006**: cero conversiones manuales de suma o resta fija de horas permanecen en el código del
  sistema tras esta corrección — el mecanismo central reemplaza cualquier ajuste manual previamente
  existente.

## Out of Scope

- **El motor de evaluación de vigencia de promociones en sí** (`active_discount_promotions`,
  `local_now`, `best_line_discount`, prioridad entre promociones empatadas) — spec 012, anomalía
  A-07, **protegida, no se toca**.
- **Las correcciones ya definidas por las specs 022 (A-08, menú/carrito QR) y 023 (A-09, POS de
  staff)** — ambas sobre la vigencia de promociones mostrada *antes* de cobrar, un problema distinto
  al de esta spec (fechas de hechos ya consumados). Si el mecanismo central que define esta spec
  llega a implementarse antes que esas dos, sus propias implementaciones deben adoptarlo, pero esta
  spec no fuerza ni reabre su alcance ya aprobado.
- **El criterio de desempate entre promociones empatadas del frontend** (A-10) — anomalía distinta.
- **La ventana de vigencia de una promoción** (`Promotion.starts_at`/`ends_at`) — hora de pared local
  recurrente, no instante absoluto (ver `FR-009`).
- **El job de expiración de promociones vencidas** (`expire_promotions`, A-39) — pertenece al ámbito
  del motor de promociones (A-07), no al de esta spec.
- **Cambiar la zona horaria del sistema operativo, del contenedor Docker o del motor de base de
  datos** — la corrección funciona de forma explícita en el mecanismo central, sin depender de ese
  ajuste de infraestructura; se documenta como hallazgo (ver Assumptions) pero no es un cambio
  requerido por esta spec.
- **Una funcionalidad de selección de zona horaria por usuario individual** — esta spec solo exige
  configuración por tenant (Historia 4), no por usuario.
- **Auditar si el desfase de horas llegó a afectar una decisión operativa real en producción antes de
  esta corrección** (por ejemplo, un cierre de caja conciliado con el día equivocado) — decisión de
  negocio no bloqueante, igual que ya se dejó para casos similares (A-07, A-08, A-09).

## Assumptions

- **Los timestamps de instante absoluto del sistema ya almacenan UTC de forma consistente**: la
  investigación de esta spec confirma que todas las columnas de fecha/hora auditadas son `DateTime`
  sin zona horaria, pobladas por `server_default=func.now()` de PostgreSQL o por
  `datetime.now(timezone.utc)` desde la aplicación, sin ningún `TZ=` distinto de UTC fijado en los
  contenedores Docker (`docker-compose.yml`/`docker-compose.prod.yml` revisados íntegros, sin esa
  variable) — de ahí que `FR-007` no exija migración de datos. Se recomienda confirmar con
  `SHOW timezone;` contra la instancia de PostgreSQL real antes de implementar, como último paso de
  verificación.
- **`America/Bogota` es la zona horaria por defecto** para el desarrollo actual y para cualquier
  tenant sin configuración explícita (Historia 4), consistente con el valor por defecto ya usado hoy
  por `TENANT_TIMEZONE` (`config.py:20`).
- **El mecanismo exacto de conversión (nombre de función/servicio, dónde vive en backend y en
  frontend, cómo se expone la zona horaria del tenant al frontend) es una decisión de diseño de la
  fase de planeación (`plan.md`)**, no de esta spec — esta spec exige que exista un único mecanismo
  reutilizado por todas las entidades listadas (`FR-002`), no prescribe su implementación.
- **Esta spec no duplica ni reabre el alcance ya aprobado de las specs 022 y 023**: ambas corrigen la
  vigencia de promociones mostrada antes de cobrar (un problema de evaluación de reglas de negocio en
  tiempo real), mientras esta spec corrige cómo se muestra la fecha/hora de un hecho ya consumado
  (una venta, un pago, un movimiento) — son necesidades de negocio distintas, aunque comparten la
  causa raíz general (falta de conversión explícita de zona horaria) y podrían terminar reutilizando
  el mismo mecanismo central si la secuencia de implementación lo permite.
- **La reapertura de A-46 y el registro de la nueva anomalía (`FR-011`) se documentan como parte de
  esta spec, no como una nueva ronda de entrevista de negocio**: el propietario del repositorio
  (deimerhdz21@gmail.com) decide corregir el defecto siguiendo el mismo criterio de riesgo latente sin
  corregir ya aplicado a A-08 y A-09.
- **Los nombres de entidad, archivo y línea citados son literales del código al momento de esta
  extracción** (2026-08-24, `pos-backend/app/models/*.py`, `pos-backend/app/api/v1/**`,
  `pos-heladeria/src/app/modules/**`). Si el código cambia antes de la implementación, esta spec debe
  re-verificarse antes de usarse como base de planeación.
