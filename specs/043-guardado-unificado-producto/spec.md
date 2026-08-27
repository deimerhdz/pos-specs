# Feature Specification: Guardado Unificado de Producto (Crear y Actualizar)

**Feature Branch**: `043-guardado-unificado-producto`

**Created**: 2026-08-27

**Status**: Draft

**Naturaleza de esta spec**: funcionalidad **nueva** de arquitectura/rendimiento sobre el
formulario de administración de productos (fase de evolución funcional, Principio I de la
[Constitución](../../.specify/memory/constitution.md)). No reabre ninguna regla de negocio de
catálogo ya definida en las specs [002](../002-catalogo-productos-variantes-y-precios/spec.md)
(precio, SKU, soft-delete, unicidad de nombre de variante), [003](../003-consumo-de-inventario-por-receta-y-opcion/spec.md)
(receta), [004](../004-validacion-grupos-opciones/spec.md) (grupos de opciones) ni
[042](../042-orden-presentaciones-producto/spec.md) (orden de presentaciones) — esta spec cambia
**cuántas peticiones HTTP** hacen falta para guardar un producto completo y **cómo se agrupan**,
no ninguna regla de validación existente sobre esos datos. Sí implica retirar los endpoints
separados que hoy usa el formulario para variantes, receta, grupos de opciones y reordenamiento
(Principio II — comportamiento existente que cambia por decisión de negocio explícita, ver
Autorización de negocio).

**Autorización de negocio (Principio I y Principio II de la
[Constitución](../../.specify/memory/constitution.md))**: solicitado directamente por el
dueño/desarrollador del proyecto el 2026-08-27, junto con las decisiones de alcance, atomicidad y
retiro de endpoints resueltas en la sección de Clarifications.

**Input**: User description: "estoy viendo cuando se crea o actualiza un producto se disparan
muchas peticiones http haciendo que la pagina se demore demasiado, es posible mejorar ese flujo
para que todo se guarde o actualice atraves de un solo endpoint ya sea para actualizar o crear,
estableciendo que cada accion es un endpoint diferente". Adjunta una captura de las herramientas de
desarrollador del navegador mostrando, para una sola acción de guardado, múltiples peticiones
separadas: `variants?active=true` (dos veces), `recipe` (dos veces), `option-groups` (dos veces),
y `reorder`.

## Clarifications

### Session 2026-08-27

- Q: ¿Qué debe incluir el guardado consolidado (el endpoint único de crear y el de actualizar)? →
  A: Todo lo que hoy se ve en el formulario de producto: datos propios del producto, presentaciones/
  variantes (crear, editar, activar/desactivar), receta por variante, asociación de grupos de
  opciones por variante, y el orden de las presentaciones — exactamente lo que hoy dispara las
  peticiones separadas `variants`, `recipe`, `option-groups` y `reorder` vistas en la captura.
- Q: Si al guardar todo junto una parte falla su validación (p. ej. una variante con nombre
  duplicado, un precio negativo), ¿qué debe pasar? → A: Todo o nada (transaccional) — si cualquier
  parte falla, no se guarda nada del envío, y el administrador recibe un único error con el detalle
  de qué parte falló.
- Q: ¿Qué pasa con los endpoints actuales por separado (variantes, receta, grupos de opciones,
  reordenamiento) una vez exista el endpoint unificado? → A: Se retiran por completo una vez
  migrado el formulario de administración de productos, previa verificación de que ningún otro
  consumidor los necesita (ver FR-007).
- Q: ¿En cuántos segundos como máximo debe completarse el guardado consolidado para un producto
  típico (hasta 10 presentaciones), además de la reducción relativa del 50% ya definida en SC-002?
  → A: Menos de 2 segundos — estándar de UX para una acción de guardado en un formulario web, sin
  romper el flujo del administrador.
- Q: ¿Debe existir un número máximo de presentaciones por producto que el guardado consolidado
  tenga que soportar garantizado, distinto del que ya soporta el sistema hoy? → A: No, sin tope
  nuevo — se soporta el mismo rango que ya permite el sistema hoy; "hasta 10" es solo el caso
  típico medido en SC-001/SC-002, no un límite duro.
- Q: Si el administrador presiona "Guardar" dos veces seguidas antes de que la primera petición
  termine, ¿qué debe pasar? → A: El formulario bloquea el botón "Guardar" mientras la primera
  petición está en curso, evitando que se dispare una segunda — el backend no necesita manejar
  duplicados porque el cliente los previene.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Crear un producto completo en un solo guardado (Priority: P1)

Un administrador abre el formulario de "nuevo producto", completa sus datos, agrega varias
presentaciones con su precio, define la receta de cada una, asocia sus grupos de opciones y
ajusta el orden en que se mostrarán — todo dentro del mismo formulario, sin salir de él — y
presiona "Guardar". Hoy esa única acción dispara una petición por cada presentación, una por cada
receta, una por cada grupo de opciones y una de reordenamiento, lo que hace que la página se sienta
lenta. Con esta funcionalidad, esa misma acción se resuelve con una sola petición de escritura al
backend.

**Why this priority**: es el escenario que reportó el usuario — crear un producto con varias
presentaciones es hoy la operación más lenta del formulario, precisamente porque multiplica
peticiones por cada presentación agregada.

**Independent Test**: se puede probar completamente creando un producto con al menos dos
presentaciones, cada una con receta y un grupo de opciones asociado, y verificando en las
herramientas de desarrollador del navegador que solo se realiza una petición de escritura al
backend (además de la carga inicial del formulario) para completar el guardado, y que el producto
resultante tiene sus presentaciones, receta, grupos de opciones y orden completos tras esa única
petición.

**Acceptance Scenarios**:

1. **Given** un formulario de producto nuevo con dos presentaciones, cada una con receta y un
   grupo de opciones asociado, **When** el administrador presiona "Guardar", **Then** el cliente
   realiza una sola petición de escritura al backend, y el producto queda creado con sus dos
   presentaciones, su receta y sus grupos de opciones asociados, en el orden en que fueron
   agregadas.
2. **Given** un formulario de producto nuevo sin ninguna presentación definida manualmente,
   **When** el administrador presiona "Guardar", **Then** el producto se crea con su presentación
   "Single" a precio 0 generada automáticamente, igual que hoy (`RN-CAT-05`, spec 002), dentro de
   la misma petición consolidada.
3. **Given** el mismo formulario, **When** la petición de guardado se completa con éxito, **Then**
   la respuesta trae el estado completo y final del producto (presentaciones, receta, grupos de
   opciones y orden), sin que el formulario necesite una petición adicional de lectura para
   mostrarlo.

---

### User Story 2 - Editar un producto existente combinando varios cambios en un solo guardado (Priority: P1)

Un administrador abre un producto ya existente con varias presentaciones, y en una misma sesión de
edición: cambia el nombre del producto, agrega una presentación nueva, edita el precio de otra,
agrega un ítem de receta a una tercera, cambia los grupos de opciones asociados a otra, y arrastra
para reordenar la lista (spec 042). Al presionar "Guardar", todos esos cambios —sin importar a
cuántas presentaciones o sub-recursos distintos afecten— se persisten con una sola petición de
escritura.

**Why this priority**: es el escenario de uso diario más frecuente del catálogo — editar un
producto existente casi siempre toca varias presentaciones a la vez, que es exactamente donde hoy
se multiplican las peticiones.

**Independent Test**: se puede probar completamente abriendo un producto con al menos tres
presentaciones, modificando datos del producto, de al menos dos presentaciones distintas (incluida
receta y grupos de opciones) y el orden, presionando "Guardar", y verificando que una sola petición
de escritura refleja todos esos cambios al recargar el formulario.

**Acceptance Scenarios**:

1. **Given** un producto existente con tres presentaciones, **When** el administrador edita el
   nombre del producto, el precio de una presentación, la receta de otra, y arrastra para reordenar
   las tres, y presiona "Guardar", **Then** el cliente realiza una sola petición de escritura al
   backend, y al recargar el formulario todos esos cambios están reflejados.
2. **Given** una presentación previamente desactivada (soft-delete, `RN-CAT-09`), **When** el
   administrador la reactiva dentro de la misma sesión de edición junto con otros cambios y
   presiona "Guardar", **Then** la reactivación se persiste en la misma petición consolidada,
   conservando su receta (spec 002 FR-010) — su orden pasa a depender de la posición en que quedó
   dentro de la lista de presentaciones enviada, igual que cualquier otra entrada del guardado
   consolidado (nota abajo sobre spec 042 FR-008).
3. **Given** una presentación nueva agregada durante la edición, con receta y grupos de opciones
   definidos antes de guardar, **When** se presiona "Guardar", **Then** la presentación, su receta
   y sus grupos de opciones se crean juntos en la misma petición, sin que el formulario necesite
   guardar primero la presentación por separado para obtener un identificador antes de poder enviar
   su receta.

---

### User Story 3 - Un error en cualquier parte del guardado no deja el producto a medias (Priority: P1)

Un administrador edita varias presentaciones de un producto a la vez y, sin darse cuenta, deja una
de ellas con un nombre que ya usa otra presentación del mismo producto (`RN-CAT-08`). Al presionar
"Guardar", el sistema rechaza el guardado completo — nada de lo editado se persiste, ni siquiera
los cambios válidos de las demás presentaciones — y el administrador recibe un único mensaje de
error que identifica cuál presentación causó el problema.

**Why this priority**: sin esta garantía, consolidar varias escrituras en una sola petición podría
introducir un riesgo nuevo que hoy no existe de forma tan directa: un guardado que persiste unas
partes y falla otras, dejando el producto en un estado intermedio difícil de razonar para el
administrador.

**Independent Test**: se puede probar completamente editando varias presentaciones válidas junto
con una que viole una regla existente (nombre duplicado, precio negativo, receta inválida, o
similar) en la misma sesión de edición, presionando "Guardar", y verificando que ninguno de los
cambios —ni siquiera los válidos— quedó persistido, y que el error devuelto identifica la parte que
falló.

**Acceptance Scenarios**:

1. **Given** una edición que incluye tres presentaciones válidas y una con nombre duplicado,
   **When** se presiona "Guardar", **Then** el backend rechaza el guardado completo con un error
   que identifica la presentación en conflicto, y ninguna de las cuatro presentaciones queda
   modificada.
2. **Given** el mismo escenario, **When** el administrador corrige el nombre duplicado y vuelve a
   presionar "Guardar", **Then** esta vez el guardado se completa con éxito y las cuatro
   presentaciones quedan persistidas.
3. **Given** una edición con un precio negativo en la receta de una presentación (fuera de las
   reglas ya vigentes en `RN-CAT-03`/`RN-CAT-04`), **When** se presiona "Guardar", **Then** el
   rechazo aplica al guardado completo, igual que en el escenario 1 — no importa en qué sub-recurso
   ocurra el error de validación.

---

### User Story 4 - Retiro de los endpoints separados una vez migrado el formulario (Priority: P2)

Una vez el formulario de administración de productos usa exclusivamente los dos endpoints
consolidados (creación y actualización), los endpoints separados que antes usaba para crear/editar
una presentación individual, su receta, sus grupos de opciones y el reordenamiento dejan de
existir, siempre que se haya verificado primero que ningún otro flujo del sistema los necesita.

**Why this priority**: es la limpieza que cierra el cambio — mantener ambos caminos (el nuevo
consolidado y el viejo granular) indefinidamente reintroduce la complejidad que esta spec busca
eliminar, pero no bloquea el valor principal (P1-P3) si se completa después.

**Independent Test**: se puede probar verificando que, tras la migración, ninguna ruta de los
endpoints separados de variante individual, receta, grupos de opciones y reordenamiento sigue
registrada ni es alcanzable, y que ningún flujo del sistema (formulario de productos, Menú QR, u
otro) depende de ellos.

**Acceptance Scenarios**:

1. **Given** el formulario de administración de productos ya migrado a los dos endpoints
   consolidados, **When** se audita el resto del sistema en busca de otros consumidores de los
   endpoints separados de variante, receta, grupos de opciones y reordenamiento, **Then** no se
   encuentra ninguno.
2. **Given** esa verificación completada sin hallazgos, **When** se retiran los endpoints
   separados, **Then** el formulario de administración de productos sigue funcionando sin ninguna
   regresión, usando únicamente los dos endpoints consolidados.
3. **Given** que la auditoría del escenario 1 sí encuentra otro consumidor de alguno de esos
   endpoints, **When** se decide el retiro, **Then** ese endpoint puntual queda documentado como
   excepción fuera del alcance de retiro de esta spec, en vez de eliminarse igual.

---

### Edge Cases

- **Hallazgo real de la auditoría de FR-007 (User Story 4, Acceptance Scenario 3)**: al
  implementar esta spec, la auditoría encontró que `app/scripts/test_variantes_duplicadas.py`
  (el characterization script citado por la spec 002 para `RN-CAT-08`/`RN-CAT-09`) importa y
  llama directamente las funciones de `POST /products/{id}/variants` y `PATCH /variants/{id}`
  (no vía HTTP, en proceso) — un consumidor real, distinto del formulario de administración de
  productos. Por eso, de los cinco endpoints candidatos a retiro, estos dos quedan **excluidos**
  como excepción documentada (igual que prevé el escenario 3 de esta historia); los otros tres
  (`PUT /variants/{id}/recipe`, `PUT /variants/{id}/option-groups`,
  `PATCH /products/{id}/variants/reorder`) sí se retiran, al no tener ningún otro consumidor
  identificado.
- **Producto nuevo sin ninguna presentación definida manualmente**: sigue aplicando la creación
  automática de la presentación "Single" a precio 0 (`RN-CAT-05`), dentro de la misma petición
  consolidada, sin ningún paso adicional.
- **SKU duplicado contra una variante de otro producto** (`RN-CAT-11`) detectado dentro de un
  guardado que además incluye otras presentaciones válidas: rechaza el guardado completo (User
  Story 3), no solo la presentación en conflicto.
- **Carrera entre dos administradores guardando el mismo producto casi al mismo tiempo**: se
  mantiene el mismo criterio ya documentado (el último guardado exitoso en llegar prevalece, spec
  002 US5 escenario 5, spec 042 Edge Cases), ahora aplicado a la transacción consolidada completa
  en vez de a una fila individual.
- **Producto con muchas presentaciones, cada una con receta extensa y varios grupos de opciones**:
  no existe un tope nuevo al número de presentaciones que el guardado consolidado deba soportar —
  se soporta el mismo rango que ya permite el sistema hoy. El objetivo medible de tiempo de
  respuesta (SC-002, menor a 2 segundos) se define sobre el caso típico de hasta 10 presentaciones,
  no como límite duro para productos más grandes.
- **Falla de red o del servidor a mitad de la única petición de guardado**: el administrador ve un
  error igual de identificable como el de una falla de validación (User Story 3); dado que todo se
  guarda en una sola transacción, no queda ningún cambio a medias por esta causa tampoco.
- **Doble clic en "Guardar" antes de que la primera petición termine**: el formulario bloquea el
  botón mientras la petición está en curso, de modo que nunca se disparan dos guardados
  concurrentes del mismo producto desde el mismo formulario (FR-008).
- **Intentar renombrar o crear una presentación con un nombre que ocupa una presentación
  desactivada del mismo producto** (`RN-CAT-08`/`RN-CAT-09`, spec 002 US5): el guardado
  consolidado completo se rechaza (User Story 3) identificando la presentación desactivada que
  ocupa el nombre, igual que hoy. La diferencia frente a hoy es **cómo se recupera** el
  administrador: como el retiro de endpoints (FR-007) alcanza también al que hoy reactiva una
  presentación de forma instantánea con una llamada aparte, la reactivación pasa a resolverse
  dentro del mismo formulario (sin llamada de red) y a incluirse en el siguiente guardado
  consolidado — el administrador ya no necesita una acción de red separada para desatascar el
  guardado, solo reintentar "Guardar" con la reactivación ya reflejada en el formulario.
- **El orden de una presentación reactivada ya no se "congela" en su valor previo a
  desactivarse** (a diferencia de `spec 042 FR-008`, escrito para la reactivación instantánea y
  aislada de antes de esta spec): dentro del guardado consolidado, el orden de **todas** las
  presentaciones —incluida una recién reactivada— se recalcula según su posición en la lista
  `variants` enviada (FR-002), la misma regla para todas las entradas sin excepción. En la práctica
  esto normalmente ubica a la reactivada al final (mismo tratamiento que agregar una presentación
  nueva, `RN-CAT-05`/FR-005 de spec 042), salvo que el formulario la reordene explícitamente antes
  de guardar. Su receta y sus grupos de opciones sí se conservan, porque el formulario los carga y
  los reenvía tal cual (Assumptions).

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: El sistema DEBE ofrecer un endpoint de creación que persista, en una sola operación
  de escritura, el producto nuevo junto con todas sus presentaciones iniciales (incluida la
  presentación "Single" automática cuando no se define ninguna, `RN-CAT-05`), la receta de cada
  presentación, la asociación de grupos de opciones de cada presentación, y el orden inicial de las
  presentaciones.
- **FR-002**: El sistema DEBE ofrecer un endpoint de actualización que persista, en una sola
  operación de escritura, todos los cambios de una sesión de edición de un producto existente:
  datos propios del producto, presentaciones nuevas, presentaciones editadas, presentaciones
  activadas o desactivadas, cambios de receta por presentación, cambios de grupos de opciones por
  presentación, y el orden resultante de las presentaciones.
- **FR-003**: El endpoint de creación y el endpoint de actualización DEBEN ser dos endpoints
  distintos, uno por acción — ninguno de los dos infiere la acción a partir de la presencia o
  ausencia de un identificador en el cuerpo de la petición.
- **FR-004**: Ante cualquier error de validación en cualquier parte del payload consolidado (por
  ejemplo, nombre de presentación duplicado `RN-CAT-08`, precio negativo de presentación u opción
  `RN-CAT-03`/`RN-CAT-04`, SKU duplicado `RN-CAT-11`, o cualquier otra regla vigente de receta o
  grupos de opciones), el sistema NO DEBE persistir ningún cambio de ese guardado — todo o nada, en
  una única transacción. La respuesta de error DEBE identificar qué parte específica del payload
  falló (p. ej. qué presentación, qué campo), para que el formulario pueda señalarlo.
- **FR-005**: Todas las reglas de negocio ya vigentes para presentaciones, receta, grupos de
  opciones y orden (specs 002, 003, 004 y 042) DEBEN seguir aplicándose sin cambios dentro del
  guardado consolidado — esta spec cambia cuántas peticiones HTTP hacen falta y cómo se agrupan, no
  ninguna regla de validación existente sobre esos datos.
- **FR-006**: La respuesta exitosa del endpoint de creación y del de actualización DEBE incluir el
  estado completo y final del producto guardado (presentaciones, receta, grupos de opciones
  asociados y orden), de forma que el formulario no necesite una petición adicional de lectura para
  reflejar lo guardado.
- **FR-007**: Una vez el formulario de administración de productos migre a usar exclusivamente los
  dos endpoints consolidados (FR-001, FR-002), los endpoints separados que hoy usa ese formulario
  para crear/editar una presentación individual, su receta, sus grupos de opciones asociados, y el
  reordenamiento DEBEN retirarse, previa verificación de que ningún otro flujo del sistema depende
  de ellos. Cualquier endpoint separado que sí tenga otro consumidor identificado queda excluido de
  este retiro y debe documentarse como excepción.
- **FR-008**: El formulario de administración de productos DEBE deshabilitar el botón "Guardar"
  mientras la petición de guardado consolidado esté en curso, de forma que un doble clic u otra
  interacción repetida no dispare una segunda petición de guardado concurrente para el mismo
  producto.

### Key Entities *(include if feature involves data)*

- **Product**: producto del menú. Entidad ya existente (spec 002); esta spec no le agrega ni le
  quita atributos, solo cambia cómo se envían sus cambios y los de sus entidades relacionadas al
  guardar.
- **ProductVariant (presentación)**: entidad ya existente (spec 002), con su receta (spec 003), sus
  grupos de opciones asociados (spec 004) y su orden de visualización (spec 042). Esta spec no
  modifica ninguno de esos atributos ni sus reglas — solo consolida cómo se crean/editan todas las
  presentaciones de un mismo producto (y sus sub-recursos) en una sola operación de escritura.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: Crear o actualizar un producto con hasta 10 presentaciones, cada una con receta y
  grupos de opciones asociados, requiere una sola petición de escritura al backend (sin contar la
  carga inicial de lectura del formulario), en vez de la petición-por-presentación, petición-por-
  receta, petición-por-grupo-de-opciones y petición-de-reordenamiento que se disparan hoy para la
  misma acción.
- **SC-002**: El tiempo entre presionar "Guardar" y ver la confirmación de guardado para un
  producto típico (hasta 10 presentaciones) se reduce en al menos 50% respecto al tiempo actual
  medido para el mismo caso, y en todo caso es **menor a 2 segundos**.
- **SC-003**: El 100% de las reglas de negocio ya vigentes citadas en esta spec (`RN-CAT-01` a
  `RN-CAT-11` de la spec 002, y las reglas de receta, grupos de opciones y orden de las specs 003,
  004 y 042) sigue cumpliéndose sin ninguna regresión observable tras consolidar el guardado.
- **SC-004**: Ante un error de validación en cualquier parte de un guardado consolidado, el
  administrador ve un único mensaje de error que identifica qué falló, y el producto conserva
  exactamente el mismo estado que tenía antes de presionar "Guardar" — en el 100% de los casos
  verificados, nunca un guardado parcial.

## Out of Scope

- **Cambiar cualquier regla de negocio de precios, SKU, receta, opciones u orden** ya definida en
  las specs 002, 003, 004 y 042 — esta spec solo cambia el transporte HTTP del guardado, no esas
  reglas.
- **Optimizar la carga inicial (lectura) del formulario de edición de producto** — esta spec cubre
  únicamente el guardado (escritura); cualquier mejora sobre las peticiones de lectura al abrir el
  formulario queda fuera de alcance.
- **Consolidar el guardado de otras entidades del catálogo** que no sean el producto y sus
  presentaciones (por ejemplo, la administración del catálogo de categorías o de grupos de
  opciones como entidades independientes, fuera del contexto de un producto puntual).
- **Retirar endpoints usados por flujos distintos al formulario de administración de productos**,
  si la verificación de FR-007 encuentra alguno — esos casos quedan documentados como excepción, no
  como parte del alcance de retiro de esta spec.

## Assumptions

- **Todas las acciones hoy disponibles desde el formulario de producto sobre sus presentaciones
  (crear, editar datos, activar/desactivar, receta, grupos de opciones, orden) se consolidan en el
  guardado único al presionar "Guardar"** — igual que ya asume, a nivel de intención de UX, la spec
  042 para el reordenamiento y la spec 027 para el toggle de manejo de inventario. Esta spec hace
  que el transporte HTTP coincida con esa intención, que hoy ya existe en el formulario pero se
  ejecuta como múltiples peticiones separadas en vez de una sola.
- **El formulario de administración de productos (`pos-heladeria`) es el único consumidor
  identificado hoy de los endpoints que se retiran (FR-007)**; si durante la implementación se
  descubre otro consumidor, ese hallazgo se documenta como excepción antes de retirar el endpoint
  correspondiente, conforme al Principio II de la Constitución.
- **El comportamiento "todo o nada" (FR-004) no revierte ninguna garantía de atomicidad existente**
  — los endpoints separados de hoy tampoco garantizan atomicidad entre sí (una falla en la petición
  N+1 no revierte las N peticiones anteriores ya persistidas); consolidar el guardado en una sola
  transacción es una mejora respecto al comportamiento actual, no un cambio que quite una garantía
  que ya existía.
- **El tamaño de "producto típico" usado en SC-001 y SC-002 (hasta 10 presentaciones) sigue el
  mismo supuesto de tamaño ya usado en spec 042** (SC-001), por consistencia entre specs del mismo
  módulo de catálogo. Esta spec no introduce ningún tope máximo nuevo de presentaciones, receta u
  opciones por producto — el objetivo de menos de 2 segundos (SC-002) se mide sobre ese caso
  típico, no como garantía para productos arbitrariamente grandes.
