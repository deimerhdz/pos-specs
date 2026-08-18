# Feature Specification: Corrección de la validación de opciones en el alta directa del mesero (A-04)

**Feature Branch**: `020-correccion-validacion-opciones-mesero`

**Created**: 2026-08-17

**Status**: Draft

**Input**: User description: "Especificación delta: corrige `add_item_to_table`
(`app/api/v1/orders/consolidation.py:199`) para que pase `variant=variant` a
`load_valid_options`, igual que ya hace `create_order` (`service.py:102`), restaurando la
validación de `min_select`/`max_select`/pertenencia de grupo de opciones (sabores/toppings) en el
único camino con botón real en la terminal del mesero. Cierra la anomalía A-04 de
`registro-de-anomalias.md`, reforzada por testimonio de negocio (P4, jefe de cocina): merma real
observada hace ~15 días en sabores/toppings. No es retroactiva — no recalcula inventario ya
consumido incorrectamente."

**Naturaleza de esta spec**: **delta de modernización**, no característica nueva. Aplica la
corrección que la spec [009-cocina-consolidacion-y-mesas-fisicas](../009-cocina-consolidacion-y-mesas-fisicas/spec.md)
(`FR-021`, User Story 1) documentó pero dejó explícitamente sin aplicar ("**Corrección**: pasar
`variant=variant` en `consolidation.py:199`"). No modifica la regla de validación en sí
(`min_select`/`max_select`/pertenencia de grupo, `RN-CAT-33`), ya especificada por completo en la
spec [004-validacion-grupos-opciones](../004-validacion-grupos-opciones/spec.md) (`FR-007`) — esta
delta corrige exclusivamente el caller que la omitía.

**Autorización de negocio (Principio I de la [Constitución](../../.specify/memory/constitution.md))**:
`registro-de-anomalias.md`, entrada A-04, clasificada **BUG HISTÓRICO CON DEPENDIENTES** con
prioridad reforzada — el único hallazgo de todo el reconocimiento con prueba directa de `git
log`/`git show` de cómo y cuándo se rompió (regresión de fusión entre la rama de corrección
`03469ca`, 2026-08-03, y la rama de combos `ee94f30`, 2026-08-04). Reforzada por
[`entrevista-negocio.md`](../000-reconocimiento/entrevista-negocio.md) P4 (jefe de cocina,
2026-08-16): confirma merma real observada en conteo físico hace ~15 días (aprox. 2026-08-01) en
sabores/toppings elegibles, coincidiendo exactamente con el patrón predicho por la reconstrucción
de código. El "Tratamiento acordado" de A-04 prioriza explícitamente su corrección en fase de
modernización (Principio V: sin recálculo retroactivo).

## User Scenarios & Testing *(mandatory)*

<!--
  Igual que la spec 009 de la que depende, esta delta cita nombres de función y argumentos porque
  son el contrato observable que se está corrigiendo, no una fuga de detalles de implementación
  (ver Assumptions).
-->

### User Story 1 - El mesero no puede agregar un ítem con sabores/opciones obligatorios incompletos (Priority: P1)

Un mesero agrega, desde el botón de la terminal, un ítem directo a la cuenta de una mesa (sin
pasar por el carrito del comensal). La variante exige elegir un número mínimo y máximo de
opciones de un grupo que descuenta inventario (p. ej. "elige 3 sabores" en una copa grande). El
mesero envía una selección incompleta o fuera de grupo. El sistema debe rechazar la operación, sin
crear el ítem ni cobrar ni descontar inventario — el mismo resultado que ya produce hoy el camino
sin caller de UI confirmado (`create_order`) ante la misma selección.

**Why this priority**: es la corrección del hallazgo de mayor evidencia y mayor impacto económico
directo de todo el reconocimiento — dinero real cobrado de más y merma de insumo confirmada por
testimonio de negocio (P4). Sin esta historia, la spec 009 sigue documentando el bug sin cerrarlo.

**Independent Test**: se puede probar de forma aislada agregando, vía el endpoint que usa
`add_item_to_table`, una variante con `min_select=3` seleccionando solo 2 opciones válidas, y
verificando que el sistema rechaza la operación con el mismo código de error que ya usa
`create_order` para el mismo caso.

**Acceptance Scenarios**:

1. **Given** una variante "Copa grande" con un grupo de opciones obligatorio `min_select=3`,
   `max_select=3`, que descuenta inventario, **When** el mesero la agrega directo a la mesa
   seleccionando solo 2 sabores válidos, **Then** el sistema rechaza la operación con `422`, no se
   crea ningún `OrderItem` y no se descuenta inventario de ningún sabor (`RN-CAT-33`).
2. **Given** la misma variante y grupo, **When** el mesero la agrega directo a la mesa
   seleccionando los 3 sabores correctos, **Then** el sistema crea el `OrderItem` normalmente,
   cobra el precio completo y descuenta inventario de los 3 sabores — mismo comportamiento que ya
   existe hoy para una selección completa.
3. **Given** la misma variante y grupo (`max_select=3`), **When** el mesero la agrega directo a la
   mesa seleccionando 4 sabores (excede el máximo permitido del grupo obligatorio), **Then** el
   sistema rechaza la operación con `422`, igual que ya hace hoy `create_order` ante la misma
   selección — el mismo mecanismo de `min_select`/`max_select` cubre tanto elegir de menos como de
   más (`RN-CAT-33`).

---

### User Story 2 - La validación es idéntica sin importar el camino de entrada (Priority: P2)

Un desarrollador de modernización necesita tener certeza de que el camino del mesero
(`add_item_to_table`) y el camino sin caller de UI confirmado (`create_order`) producen
exactamente el mismo resultado ante la misma selección de opciones, cerrando la divergencia que
documentó la spec 009 (`FR-021`).

**Why this priority**: es la historia de verificación de la corrección, no de un comportamiento
nuevo — de ahí su prioridad menor frente a la Historia 1, que sí corrige dinero e inventario mal
calculados hoy.

**Independent Test**: se puede probar de forma aislada invocando ambos caminos con el mismo
conjunto de opciones fuera de rango y comparando que ambos rechazan con el mismo código de error.

**Acceptance Scenarios**:

1. **Given** el mismo conjunto de opciones incompleto para una variante con grupo obligatorio,
   **When** se agrega vía `add_item_to_table` y, por separado, vía `create_order`, **Then** ambos
   caminos rechazan la operación con el mismo código de error (`RN-CAT-33`, cierra `FR-021` de la
   spec 009).
2. **Given** un ítem agregado por combo (`combo_id`, sin selección propia de opciones), **When**
   se agrega directo a la mesa, **Then** el sistema lo acepta sin pedir sabores, sin cambio frente
   al comportamiento de hoy — la corrección no afecta el camino de combos.

---

### User Story 3 - La corrección no altera ningún pedido ni factura ya generados (Priority: P1)

El dueño/gerente necesita la certeza de que corregir la validación hacia adelante no recalcula ni
altera ningún ítem, orden o factura ya emitidos con selección incompleta antes de esta corrección.

**Why this priority**: sin esta garantía, la corrección viola el Principio V de la constitución
(ningún cambio retroactivo) — es tan crítica como la corrección misma, no un detalle secundario.

**Independent Test**: se puede probar de forma aislada consultando, antes y después de aplicar la
corrección, un pedido/factura ya emitidos con selección incompleta y verificando que su valor no
cambia.

**Acceptance Scenarios**:

1. **Given** un `OrderItem` creado antes de esta corrección con una selección de opciones
   incompleta, **When** se consulta ese ítem o la factura que lo contiene después de aplicar la
   corrección, **Then** su valor y su inventario ya descontado permanecen exactamente iguales —
   no se recalculan ni se marcan como inválidos.

---

### Edge Cases

- ¿Qué pasa si la variante no tiene ningún grupo de opciones obligatorio? Se agrega igual que hoy,
  sin pedir nada — la corrección solo activa la validación cuando existe un grupo que la exige, no
  introduce una exigencia nueva donde no la había.
- ¿Qué pasa si el mesero reemplaza (anula y sustituye) un ítem por una variante con opciones
  obligatorias vía `void_item`? Ese camino ya pasa `variant` (`repl_variant`) y valida
  correctamente hoy — no es parte de la anomalía A-04 y no cambia con esta corrección.
- ¿Qué pasa con los ítems ya en la cuenta de una mesa agregados antes de la corrección con
  selección incompleta? Quedan igual — ver User Story 3, sin recálculo retroactivo.
- ¿Qué pasa si dos meseros agregan a la misma mesa casi al mismo tiempo? Fuera de alcance de esta
  delta — ninguno de los dos caminos toma un bloqueo de fila (`with_for_update`); es un riesgo de
  concurrencia distinto ya señalado en la spec 009 (registro de riesgos R1), no parte de A-04.
- ¿Qué pasa si la variante exige un grupo opcional (`min_select=0`) y el mesero no envía ninguna
  opción de ese grupo? Se acepta sin rechazo — la corrección solo exige cumplir `min_select`
  cuando es mayor a cero, igual que ya hace hoy `create_order`.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001 [Corrige A-04, `FR-021` de la spec 009, Historia 1]**: `add_item_to_table` DEBE pasar
  el parámetro `variant` a `load_valid_options` (`consolidation.py:199`), activando la validación
  de `min_select`/`max_select`/pertenencia de grupo sobre la selección de opciones, con el mismo
  criterio que ya aplica `create_order` (`service.py:102`).
- **FR-002 [Corrige A-04, Historia 1]**: ante una selección de opciones incompleta o fuera de
  grupo en el alta directa del mesero, el sistema DEBE rechazar la operación sin crear el
  `OrderItem` ni descontar inventario — con el mismo código de error (`422`) que ya usa
  `create_order` para el mismo caso.
- **FR-003 [Historia 2, consistencia]**: para un mismo conjunto de opciones, el resultado de
  `add_item_to_table` DEBE ser idéntico al de `create_order` — ambos aceptan o ambos rechazan la
  misma selección.
- **FR-004 [Sin cambio, se conserva]**: un ítem agregado por combo (`combo_id`, sin selección
  propia de opciones) DEBE seguir aceptándose sin validación de `min_select`/`max_select` — la
  corrección no introduce ninguna exigencia nueva sobre el camino de combos.
- **FR-005 [Principio V de la constitución, ningún cambio retroactivo, Historia 3]**: el sistema
  NO DEBE recalcular, reversar ni alterar ningún `OrderItem`, orden ni factura ya generados con
  selección de opciones incompleta antes de esta corrección.
- **FR-006 [Principio II de la constitución, trazabilidad de la corrección]**: la implementación
  DEBE incluir al menos un test de characterization dedicado que demuestre, antes de la
  corrección, la divergencia entre `add_item_to_table` y `create_order` ante la misma selección
  incompleta, y confirme, después, que ambos caminos la rechazan igual — citando en su nombre o
  comentario la anomalía A-04 y la decisión de `registro-de-anomalias.md` que autoriza el cambio.

### Key Entities

- **OrderItem**: la línea de pedido creada por `add_item_to_table`; su existencia (o rechazo antes
  de crearse) es el efecto observable de esta corrección.
- **Option / OptionGroup**: la opción elegida (sabor/topping) y el grupo que define
  `min_select`/`max_select`/pertenencia; la validación que esta delta restaura opera sobre ellos.
- **ProductVariant**: la variante del producto agregado; su relación con `OptionGroup`
  (`VariantOptionGroup`) determina si existe alguna regla que exigir.

## Success Criteria *(mandatory)*

<!--
  Igual que la spec 009 de la que depende, esta delta cita nombres de función y argumentos porque
  son el contrato observable que se está corrigiendo, no una fuga de detalles de implementación
  (ver Assumptions).
-->

### Measurable Outcomes

- **SC-001**: Con una variante cuyo grupo de opciones obligatorio exige un número específico de
  selecciones, el alta directa del mesero (`add_item_to_table`) rechaza el 100% de las selecciones
  que incumplen `min_select`/`max_select` de ese grupo (de menos o de más), sin excepción.
- **SC-002**: Para el 100% de las combinaciones de selección probadas, el resultado de
  `add_item_to_table` coincide con el de `create_order` — aceptación o rechazo, nunca divergen.
- **SC-003**: Ningún `OrderItem`, orden ni factura emitidos antes de esta corrección cambia de
  valor ni de inventario descontado como consecuencia de este cambio.
- **SC-004**: Existe al menos un script de characterization que demuestra la divergencia previa
  entre los dos caminos y su cierre tras la corrección, cerrando el gap de caracterización
  señalado como prioritario en la spec 004 (`SC-002`) para esta forma de selección.

## Out of Scope

- **La regla de validación en sí** (`min_select`/`max_select`, pertenencia de grupo,
  `validate_option_selection`, tolerancia de migración de catálogo `STRICT_OPTION_SELECTION`) —
  especificada por completo en la spec 004 (`RN-CAT-33`, `FR-007`); esta delta solo corrige el
  caller que la omitía, no redefine la regla.
- **El carrito QR del comensal** (`cart/service.py`) — ya valida correctamente hoy y no forma
  parte del camino donde vive A-04; no se toca.
- **Auditar el catálogo para dimensionar la merma acumulada** antes de esta corrección — sigue
  siendo una decisión de negocio pendiente (spec 009, `assumptions`) que no bloquea la corrección
  de la línea.
- **Recalcular o ajustar retroactivamente** inventario, pedidos o facturas ya afectados por la
  selección incompleta previa a esta corrección (ver FR-005/SC-003).
- **El resto de anomalías documentadas en la spec 009** (A-16, A-26, A-48) — defectos distintos
  sobre módulos distintos de esa misma spec; no forman parte de esta delta.
- **Cualquier cambio de UI/frontend** — la terminal del mesero ya consume el mismo endpoint; solo
  cambia la respuesta de error cuando la selección enviada es inválida.
- **El bloqueo de fila (`with_for_update`) ante escrituras concurrentes** sobre el mismo `OrderItem`
  — riesgo de código distinto, ya señalado en la spec 009 (registro de riesgos R1), no parte de la
  anomalía A-04.

## Assumptions

- **Esta es una spec delta de corrección, no de característica nueva**: al igual que las specs
  009/019, cita nombres de función y argumentos explícitamente porque son el contrato observable
  que se está corrigiendo — los criterios de aceptación deben poder verificarse directamente
  contra `pos-backend` en ejecución, antes y después del cambio.
- **La autorización de negocio ya existe y no requiere una nueva ronda de entrevista**: el
  "Tratamiento acordado" de A-04 en `registro-de-anomalias.md` más la respuesta P4 de
  `entrevista-negocio.md` (jefe de cocina, 2026-08-16) satisfacen el Principio I de la
  constitución.
- **No se requiere migración de datos**: el cambio es puramente de comportamiento en tiempo de
  escritura (`add_item_to_table` no persiste su resultado de otra forma); no hay estado existente
  que migrar, solo el cuidado de no tocar ítems/facturas ya creados (FR-005).
- **La auditoría del alcance de la merma acumulada queda como decisión de negocio pendiente,
  separada de esta corrección**: igual que ya lo dejó la spec 009, esta delta no bloquea la
  corrección de la línea a que se resuelva esa auditoría.
