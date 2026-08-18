# Feature Specification: Corrección de la cuenta de mesas fusionadas (`group_bill`)

**Feature Branch**: `019-correccion-cuenta-mesas-fusionadas`

**Created**: 2026-08-17

**Status**: Draft

**Input**: User description: "Especificación delta: corrige `tables_advanced.group_bill`
(`app/api/v1/orders/tables_advanced.py:92-114`) para que excluya órdenes `pagada`/`cancelada` y
aplique promociones/combos vigentes, igual que `table_sessions.compute_bill`. Cierra la anomalía
A-01 (camino C) de `registro-de-anomalias.md`, corrigiendo `RN-ORD-64` [DUDOSA]. No toca el
camino A (`table_sessions.compute_bill`, correcto) ni el camino B (`orders/checkout.compute_bill`,
código muerto ligado a la spec 008). Sin cambios retroactivos."

**Naturaleza de esta spec**: **delta de modernización**, no característica nueva ni ingeniería
inversa. Modifica el comportamiento observado y documentado en
[009-cocina-consolidacion-y-mesas-fisicas](../009-cocina-consolidacion-y-mesas-fisicas/spec.md)
(`RN-ORD-64`, `tables_advanced.group_bill`) para que deje de divergir de la implementación
correcta y vigente de "cuánto se le debe cobrar a una mesa", documentada tal cual (sin cambios)
en [010-sesion-mesa-reparto-cierre-barrido](../010-sesion-mesa-reparto-cierre-barrido/spec.md)
(`table_sessions.compute_bill`). Cierra la anomalía **A-01** (camino C) de
[`registro-de-anomalias.md`](../000-reconocimiento/registro-de-anomalias.md).

**Autorización de negocio (Principio I de la [Constitución](../../.specify/memory/constitution.md))**:
`registro-de-anomalias.md`, entrada A-01, "Tratamiento acordado" (2026-08-15/16) +
[`entrevista-negocio.md`](../000-reconocimiento/entrevista-negocio.md) P1 (dueño/gerente,
2026-08-16): confirma que la función de fusionar/agrupar mesas no se usa a diario hoy — el
riesgo es **latente, no activo** — pero no cambia el tratamiento ya acordado: corregir en fase
de modernización, replicando el criterio de `table_sessions.compute_bill`, sin retroactividad
(Principio V).

**Hueco cerrado de paso**: la spec 009 vigente declara en su sección "Out of Scope" que
`group_bill`/`RN-ORD-64` "sí se documenta aquí... (User Story 8, escenario relacionado)", pero
ninguno de sus 8 escenarios de aceptación ni sus `FR-001`-`FR-033` cubren en realidad el cálculo
de `group_bill` — es un hueco real entre lo que esa spec dice cubrir y lo que cubre. Esta delta
lo cierra: a partir de aquí, el comportamiento correcto de `group_bill` vive en esta spec, no en
la 009.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - La cuenta de un grupo de mesas fusionadas no vuelve a cobrar una orden ya pagada (Priority: P1)

El cajero fusiona varias mesas en un solo grupo de cobro. Una de las órdenes del grupo ya se
pagó por separado antes de fusionar (por ejemplo, vía el ciclo de cobro legado por pedido). Al
consultar la cuenta del grupo para cobrar el resto, el sistema no debe volver a incluir el monto
de la orden ya pagada.

**Why this priority**: es el defecto con mayor impacto económico directo de los dos que corrige
esta spec — duplica el cobro a un comensal, con dinero real de por medio y sin ningún rastro de
error visible para el cajero.

**Independent Test**: se puede probar de forma aislada llamando a `GET
/orders/group/{group_id}/bill` sobre un grupo con una orden `pagada` y otra `abierta`, y
verificando que el total no incluye la orden pagada — sin necesidad de que exista ninguna
promoción vigente.

**Acceptance Scenarios**:

1. **Given** un grupo de mesas fusionadas con la orden A en estado `pagada` ($20.000 en ítems no
   anulados) y la orden B en estado `abierta` ($15.000 en ítems no anulados, sin promoción
   vigente), **When** se consulta la cuenta del grupo, **Then** el total devuelto es $15.000 —
   la orden A queda excluida.
2. **Given** un grupo de mesas fusionadas con la orden A en estado `cancelada`, **When** se
   consulta la cuenta del grupo, **Then** el total devuelto excluye toda la orden A, igual que
   ya excluye hoy sus ítems `anulado` individualmente.

---

### User Story 2 - La cuenta de un grupo de mesas fusionadas refleja las promociones y combos vigentes (Priority: P1)

El cajero consulta la cuenta de un grupo de mesas fusionadas que tiene una promoción automática
vigente sobre alguna de sus líneas (por ejemplo, un descuento por categoría o un combo). El
monto mostrado debe reflejar ya el descuento, igual que lo hace hoy la cuenta de una mesa
individual sin fusionar.

**Why this priority**: mismo nivel de impacto económico directo que la Historia 1 — cobrar el
monto bruto en vez del neto le quita al comensal un descuento al que tiene derecho, sin que
quede evidencia visible del error.

**Independent Test**: se puede probar de forma aislada llamando a `GET
/orders/group/{group_id}/bill` sobre un grupo con una sola orden `abierta` que tiene una
promoción vigente aplicable, y verificando que el total ya descuenta esa promoción — sin
necesidad de que exista ninguna orden `pagada`/`cancelada` en el grupo.

**Acceptance Scenarios**:

1. **Given** un grupo de mesas fusionadas con la orden B en estado `abierta` ($15.000 brutos en
   ítems no anulados, con una promoción `percent` del 10% vigente sobre esa categoría), **When**
   se consulta la cuenta del grupo, **Then** el total devuelto es $13.500, no $15.000.
2. **Given** el mismo grupo que en el escenario 1 de la Historia 1 (orden A `pagada` $20.000,
   orden B `abierta` $15.000 brutos con 10% de descuento vigente), **When** se consulta la
   cuenta del grupo, **Then** el total devuelto es $13.500 — ambos defectos corregidos a la vez
   (ejemplo original de `contradiccion-06-cuenta-a-cobrar-tres-implementaciones.md §3`, que hoy
   devolvería $35.000).

---

### User Story 3 - La cuenta de un grupo fusionado da el mismo resultado que la cuenta de una mesa individual con las mismas órdenes (Priority: P2)

El dueño/gerente quiere una sola fuente de verdad de "cuánto se le debe cobrar", sin importar si
la mesa está sola o fusionada con otras. Esta historia no introduce ningún comportamiento nuevo
propio — es la verificación de que las Historias 1 y 2 realmente igualan el criterio de
`table_sessions.compute_bill`, no solo lo aproximan.

**Why this priority**: es una historia de verificación/consistencia, no de corrección de un
defecto nuevo — de ahí su prioridad menor frente a las dos anteriores, que sí corrigen dinero mal
calculado hoy.

**Independent Test**: se puede probar de forma aislada tomando el mismo conjunto de órdenes y
líneas, calculando la cuenta primero como si fuera una única mesa (vía
`table_sessions.compute_bill`) y luego como grupo fusionado de esa única mesa (vía
`group_bill`), y comparando que ambos resultados coincidan centavo a centavo.

**Acceptance Scenarios**:

1. **Given** una mesa individual sin fusionar con una combinación cualquiera de órdenes
   `abierta`/`pagada`/`cancelada` y promociones vigentes, **When** se calcula su cuenta por la
   vía normal (`table_sessions.compute_bill`) y, por separado, se fusiona esa misma mesa sola en
   un grupo y se calcula la cuenta del grupo (`group_bill`), **Then** ambos totales son
   idénticos.

---

### Edge Cases

- ¿Qué pasa si **todas** las órdenes del grupo están `pagada` o `cancelada`? El total debe ser
  $0 (grupo sin nada pendiente de cobro), no un error — es el mismo comportamiento que ya tiene
  `table_sessions.compute_bill` cuando no queda ninguna orden cobrable.
- ¿Qué pasa si el grupo fusionado contiene una sola mesa (fusión degenerada)? Debe dar
  exactamente el mismo resultado que la cuenta individual de esa mesa (cubierto por la Historia
  3).
- ¿Qué pasa si una orden del grupo está `cancelada` pero conserva ítems con
  `estado_cocina` distinto de `anulado` (la inconsistencia de datos que la propia `RN-ORD-64`
  original dejaba como pregunta abierta)? La orden se excluye completa por su `status`, sin
  mirar el `estado_cocina` de sus ítems individuales — el filtro por `status` de la orden tiene
  prioridad sobre el filtro por ítem.
- ¿Qué pasa si dos promociones vigentes compiten por la misma línea de un grupo fusionado? Se
  resuelve con el mismo criterio de prioridad explícita que ya usa `promotions.evaluate` para
  una mesa individual (spec 012) — esta spec no define un criterio de desempate nuevo.
- ¿Qué pasa con una cuenta de grupo ya cobrada **antes** de este cambio, calculada con el
  criterio anterior (sin excluir pagadas, sin aplicar promociones)? No se recalcula ni se
  reversa — ver Principio V de la constitución y FR-004.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001 [Corrige `RN-ORD-64` [DUDOSA], anomalía A-01 camino C, Historia 1]**: `group_bill`
  DEBE excluir del cálculo cualquier orden del grupo cuyo `status` sea `pagada` o `cancelada` —
  mismo criterio de `_billable_orders` que ya usa `table_sessions.compute_bill`.
- **FR-002 [Corrige `RN-ORD-64`, Historia 2]**: `group_bill` DEBE aplicar `promotions.evaluate`
  y `combo_discount_for_lines` sobre las líneas cobrables del grupo, con el mismo criterio de
  evaluación (hora local del tenant, prioridad explícita entre promociones empatadas — spec 012)
  que ya aplica `table_sessions.compute_bill`.
- **FR-003 [Sin cambio, se conserva]**: `group_bill` DEBE seguir excluyendo del cálculo
  cualquier ítem individual con `estado_cocina="anulado"`, sin importar el `status` de la orden
  que lo contiene — este criterio de la implementación actual ya es correcto.
- **FR-004 [Principio V de la constitución, ningún cambio retroactivo]**: el sistema NO DEBE
  recalcular, reemitir ni alterar ninguna cuenta de grupo ni factura ya emitida con el criterio
  de cálculo anterior a este cambio.
- **FR-005 [Historia 3, consistencia]**: para un mismo conjunto de órdenes y líneas, el
  resultado de `group_bill` DEBE ser idéntico, ítem por ítem y centavo por centavo, al que
  produciría `table_sessions.compute_bill` si esas mismas órdenes pertenecieran a una única mesa
  sin fusionar.
- **FR-006 [Principio II de la constitución, trazabilidad de la corrección]**: la
  implementación DEBE incluir al menos un test de characterization dedicado a `group_bill`
  (hoy inexistente — ninguno de los 12 scripts previos ni la spec 009 lo cubren) que verifique
  el comportamiento corregido de FR-001/FR-002/FR-003, citando en su nombre o comentario la
  anomalía A-01 y la decisión de `registro-de-anomalias.md` que autoriza el cambio.

### Key Entities

- **Grupo de mesas fusionadas (`merged_group_id`)**: agrupa varias `CustomerOrder` de mesas
  distintas bajo un mismo identificador de cobro conjunto. No es una tabla propia — es un campo
  compartido entre las órdenes del grupo.
- **CustomerOrder**: la orden individual dentro del grupo; su `status` (`abierta`, `bloqueada`,
  `pagada`, `cancelada`) determina si se incluye o no en el total (FR-001).
- **OrderItem**: la línea dentro de una orden; su `estado_cocina` (`anulado` o no) determina si
  se incluye o no en el total, independientemente del `status` de la orden (FR-003).

## Success Criteria *(mandatory)*

<!--
  Esta spec, igual que las specs 009/010/018 de las que depende, es una corrección sobre
  comportamiento ya observado y documentado con evidencia de código — los criterios citan
  endpoints y nombres de función porque son el contrato observable que se está corrigiendo, no
  una fuga de detalles de implementación (ver Assumptions).
-->

### Measurable Outcomes

- **SC-001**: Con un grupo de mesas fusionadas que incluye al menos una orden `pagada`, el monto
  devuelto por `GET /orders/group/{group_id}/bill` nunca incluye las líneas de esa orden.
- **SC-002**: Con un grupo de mesas fusionadas que tiene una promoción o combo vigente sobre
  alguna línea, el monto devuelto por `GET /orders/group/{group_id}/bill` siempre refleja el
  descuento correspondiente.
- **SC-003**: Para el 100% de las combinaciones de órdenes/líneas probadas, el monto de la
  cuenta de un grupo fusionado de una sola mesa coincide exactamente con el monto de la cuenta
  de esa misma mesa sin fusionar.
- **SC-004**: Ninguna cuenta de grupo ni factura emitida antes de este cambio cambia de valor
  como consecuencia de esta corrección.
- **SC-005**: El ejemplo cuantitativo documentado en la anomalía A-01
  (`contradiccion-06-cuenta-a-cobrar-tres-implementaciones.md §3`: grupo con orden pagada de
  $20.000 y orden abierta de $15.000 brutos con 10% de descuento) pasa de devolver $35.000 a
  devolver $13.500.

## Out of Scope

- **El camino B** (`orders/checkout.compute_bill`, código muerto sin caller conocido) — su
  retiro o unificación queda ligado a la resolución del ciclo `pay_order` legado, cubierto por
  la spec 008 (ya con testimonio de negocio de que ese ciclo no se usa — P11 de
  `entrevista-negocio.md`); no se toca en esta delta para no mezclar dos módulos a la vez
  (Principio III).
- **El camino A** (`table_sessions.compute_bill`) — es la referencia correcta y vigente; esta
  spec no le introduce ningún cambio, solo lo usa como criterio a replicar.
- **El no determinismo de `merge_orders`** al fusionar grupos preexistentes en colisión
  (`RN-ORD-63`, anomalía A-26) — es un defecto distinto sobre una función distinta
  (`tables_advanced.merge_orders`, no `group_bill`), ya tratado en la propia spec 009.
- **Cualquier cambio de UI/frontend** — esta delta es de cálculo en backend; la pantalla de
  cobro de grupo (`table.service.ts:133`) ya consume el endpoint existente (`GET
  /orders/group/{group_id}/bill`) sin cambios de contrato de respuesta.
- **Recalcular o auditar cuentas de grupo ya cobradas** antes de este cambio (ver FR-004/SC-004).
- **La decisión de si vale la pena seguir ofreciendo la función de fusionar mesas** — fuera de
  alcance de esta spec; el negocio ya confirmó (P1) que no se usa a diario hoy, pero el código
  sigue expuesto y esta spec solo corrige su cálculo, no decide su futuro.

## Assumptions

- **Esta es una spec delta de corrección, no de característica nueva**: al igual que las specs
  009/010/018, cita endpoints, nombres de función y valores literales explícitamente porque son
  el contrato observable que se está corrigiendo — los criterios de aceptación deben poder
  verificarse directamente contra `pos-backend` en ejecución, antes y después del cambio.
- **La autorización de negocio ya existe y no requiere una nueva ronda de entrevista**: el
  "Tratamiento acordado" de A-01 en `registro-de-anomalias.md` más la respuesta P1 de
  `entrevista-negocio.md` (dueño/gerente, 2026-08-16) satisfacen el Principio I de la
  constitución — confirman que el riesgo es latente (no urgente) pero no retiran el mandato de
  corregirlo.
- **El criterio de desempate entre promociones empatadas no se redefine aquí**: se asume el ya
  definido y vigente para `table_sessions.compute_bill` (prioridad explícita, spec 012), sin
  introducir una regla nueva propia de `group_bill`.
- **No se requiere migración de datos**: el cambio es puramente de cálculo en tiempo de
  consulta (`group_bill` no persiste su resultado), así que no hay estado existente que migrar,
  solo el cuidado de no tocar facturas/cuentas ya cerradas (FR-004).
