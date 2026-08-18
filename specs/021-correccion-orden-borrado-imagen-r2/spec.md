# Feature Specification: Corrección del orden de borrado de imagen en R2 (A-44)

**Feature Branch**: `021-correccion-orden-borrado-imagen-r2`

**Created**: 2026-08-18

**Status**: Draft

**Input**: User description: "Especificación delta: corrige `update_product`
(`app/api/v1/products/service.py:67-91`) para que el borrado del objeto de imagen anterior en
Cloudflare R2 ocurra DESPUÉS de un `db.commit()` exitoso, en lugar de antes como ocurre hoy. Cierra
la anomalía A-44 de `registro-de-anomalias.md` (ACCIDENTAL, `registro-riesgos.md` R23, severidad
Baja): si el commit falla tras el borrado actual, `product.image_url` revierte a la URL vieja pero
el objeto que señala ya fue borrado, dejando el producto con una referencia de imagen rota sin
ningún registro. No es retroactiva — no recalcula ni corrige productos ya afectados por este orden
antes de la corrección."

**Naturaleza de esta spec**: **delta de modernización**, no característica nueva. Corrige el
`FR-012` / User Story 7 de la spec [002-catalogo-productos-variantes-y-precios](../002-catalogo-productos-variantes-y-precios/spec.md),
que documentó el orden actual de operaciones (borrado en R2 antes del commit) tal cual, dejando
explícitamente la corrección para la fase de modernización ("se documenta el orden actual... sin
implementar la corrección acordada"). No modifica ninguna otra regla de esa spec (`RN-CAT-01` a
`RN-CAT-11`), solo el orden de operaciones de `update_product` ante un cambio de `image_url`.

**Autorización de negocio (Principio I de la [Constitución](../../.specify/memory/constitution.md))**:
`registro-de-anomalias.md`, entrada A-44, clasificada **ACCIDENTAL** (verificable por código,
`registro-riesgos.md` R23, severidad Baja). "Tratamiento acordado": corregir en fase de
modernización invirtiendo el orden (commit primero, borrado después) o moviendo el borrado a un
proceso asíncrono post-commit. Esta delta aplica la primera alternativa — inversión síncrona del
orden — por ser la más simple y de menor riesgo, sin afectar ningún dato retroactivamente
(Principio V).

## User Scenarios & Testing *(mandatory)*

<!--
  Igual que las specs 002/019/020 de las que depende, esta delta cita nombres de función y línea
  porque son el contrato observable que se está corrigiendo, no una fuga de detalles de
  implementación (ver Assumptions).
-->

### User Story 1 - Un fallo de guardado no deja al producto sin ninguna imagen válida (Priority: P1) — anomalía A-44

Un administrador reemplaza la foto de un producto ya existente desde el panel/API
(`PATCH /products/{id}` con `image_url` distinto al actual). El guardado del cambio falla por
cualquier razón no relacionada con la imagen (p. ej. un error de base de datos). El sistema no debe
haber borrado la imagen anterior en Cloudflare R2 — el producto debe seguir teniendo una imagen
válida y recuperable.

**Why this priority**: es el defecto de integridad de datos que expone A-44 — sin esta corrección,
el caso raro de fallo de guardado deja un producto con una referencia de imagen rota y sin ningún
registro de que ocurrió.

**Independent Test**: se puede probar de forma aislada llamando `PATCH /products/{id}` con un
`image_url` nuevo sobre un producto que ya tiene imagen, forzando un fallo en el guardado posterior
al intento de cambio, y verificando que el objeto de imagen anterior sigue existiendo en el
almacenamiento.

**Acceptance Scenarios**:

1. **Given** un producto con `image_url` apuntando a un objeto existente en R2, **When**
   `PATCH /products/{id}` trae un `image_url` distinto y el guardado posterior falla por cualquier
   razón no relacionada con la imagen, **Then** el objeto de imagen anterior sigue existiendo en R2
   y `product.image_url` en base de datos sigue apuntando a él — nunca queda una referencia rota
   (`registro-riesgos.md` R23).
2. **Given** el mismo escenario de fallo, **When** se consulta el producto después del intento
   fallido, **Then** su `image_url` es idéntico al que tenía antes del intento — el rollback no deja
   ninguna inconsistencia entre base de datos y almacenamiento.

---

### User Story 2 - El camino feliz produce el mismo resultado final que hoy (Priority: P2)

Un administrador reemplaza la foto de un producto y el guardado tiene éxito. El resultado final
debe ser idéntico al comportamiento actual: la imagen nueva queda persistida y la imagen anterior
queda eliminada del almacenamiento.

**Why this priority**: esta corrección solo debe cambiar el *orden* de las operaciones ante un
fallo; no debe alterar el resultado observable del camino feliz, que ya funciona correctamente hoy.

**Independent Test**: se puede probar de forma aislada llamando `PATCH /products/{id}` con un
`image_url` nuevo sobre un producto que ya tiene imagen, dejando que el guardado tenga éxito, y
verificando que la imagen anterior ya no existe en R2 y la nueva sí.

**Acceptance Scenarios**:

1. **Given** un producto con `image_url` apuntando a un objeto existente en R2, **When**
   `PATCH /products/{id}` trae un `image_url` distinto y el `db.commit()` tiene éxito, **Then** el
   sistema borra el objeto de imagen anterior en R2 y `product.image_url` en base de datos apunta a
   la imagen nueva — mismo resultado final que produce hoy el camino feliz.
2. **Given** el mismo escenario de éxito, **When** el borrado del objeto anterior en R2 falla por
   una razón ajena (p. ej. R2 no disponible en ese instante), **Then** el cambio de imagen ya
   confirmado en base de datos no se revierte — el borrado sigue siendo best-effort, igual que hoy.

---

### User Story 3 - La corrección no altera ningún producto ya actualizado antes del cambio (Priority: P1)

El dueño/gerente necesita la certeza de que corregir el orden de operaciones hacia adelante no
recalcula ni altera ningún producto cuya imagen ya fue actualizada (correcta o incorrectamente)
antes de esta corrección.

**Why this priority**: sin esta garantía, la corrección viola el Principio V de la constitución
(ningún cambio retroactivo) — es tan crítica como la corrección misma.

**Independent Test**: se puede probar de forma aislada consultando, antes y después de aplicar la
corrección, un producto ya actualizado antes del cambio y verificando que su `image_url` no varía.

**Acceptance Scenarios**:

1. **Given** un producto cuya imagen fue actualizada antes de esta corrección, **When** se consulta
   ese producto después de aplicar la corrección, **Then** su `image_url` y el estado del objeto en
   R2 permanecen exactamente iguales — no se recalculan ni se marcan como inválidos.

---

### Edge Cases

- ¿Qué pasa si la imagen anterior no pertenece al bucket configurado (URL externa ajena)? Igual que
  hoy: `key_from_public_url` devuelve `None` y no se intenta borrar nada, sin importar el orden — sin
  cambio frente al comportamiento actual.
- ¿Qué pasa si el producto no tenía ninguna imagen previa (primera carga, `old_image_url` vacío)? No
  hay objeto que borrar; el comportamiento no cambia.
- ¿Qué pasa si dos actualizaciones de imagen del mismo producto ocurren casi al mismo tiempo? Fuera
  de alcance de esta delta — riesgo de concurrencia distinto, ya señalado en la spec 002, no forma
  parte de la anomalía A-44.
- ¿Qué pasa si el borrado del objeto viejo falla después de un commit exitoso? El producto queda
  correcto en base de datos (apunta a la imagen nueva); el objeto viejo queda huérfano en el
  almacenamiento — mismo riesgo residual "best-effort" que existe hoy; esta corrección no lo agrava
  ni lo resuelve, solo elimina el caso de referencia rota que expone A-44.
- ¿Qué pasa con los productos cuya imagen ya quedó con una referencia rota por el orden antiguo antes
  de esta corrección? Quedan igual — ver User Story 3, sin recálculo retroactivo ni reparación
  automática.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001 [Corrige A-44, `FR-012` de la spec 002, Historia 1]**: `update_product` DEBE confirmar
  (persistir con éxito, `db.commit()`) el cambio de `image_url` de un producto **antes** de borrar
  el objeto de imagen anterior en Cloudflare R2 — invirtiendo el orden actual
  (`app/api/v1/products/service.py:67-91`).
- **FR-002 [Corrige A-44, Historia 1]**: si la persistencia del cambio de imagen falla, el sistema NO
  DEBE borrar el objeto de imagen anterior — el objeto en R2 y `product.image_url` en base de datos
  DEBEN quedar consistentes entre sí (ambos apuntando a la imagen vieja).
- **FR-003 [Historia 2, sin cambio en el camino feliz]**: si la persistencia del cambio de imagen
  tiene éxito, el sistema DEBE borrar el objeto de imagen anterior en R2 — mismo resultado final que
  produce hoy el camino feliz.
- **FR-004 [Sin cambio, se conserva]**: el borrado del objeto anterior DEBE seguir siendo
  best-effort — un fallo al borrar (`delete_object`) NO DEBE revertir ni afectar el cambio de imagen
  ya persistido, igual que hoy.
- **FR-005 [Principio V de la constitución, ningún cambio retroactivo, Historia 3]**: el sistema NO
  DEBE recalcular, revertir ni alterar ningún `Product` cuya imagen ya haya sido actualizada antes de
  esta corrección.
- **FR-006 [Principio II de la constitución, trazabilidad de la corrección]**: la implementación
  DEBE incluir al menos un test de characterization que demuestre, antes de la corrección, el orden
  actual (borrado antes del commit) y, después, el nuevo orden (borrado después del commit) —
  citando en su nombre o comentario la anomalía A-44 y la decisión de `registro-de-anomalias.md` que
  autoriza el cambio.

### Key Entities

- **Product**: producto del menú cuya imagen se actualiza; su atributo `image_url` es el efecto
  observable directo de esta corrección.
- **Objeto en Cloudflare R2**: archivo de imagen del producto, identificado por una `key` derivada
  de `image_url` (`key_from_public_url`). Su ciclo de vida (subida, borrado) es externo a la base de
  datos transaccional del sistema — origen de la anomalía A-44 que esta delta corrige.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: El 100% de los intentos de actualización de imagen que fallan en el guardado
  posterior deja el objeto de imagen anterior intacto en R2 y `product.image_url` sin cambios — sin
  excepción.
- **SC-002**: El 100% de las actualizaciones de imagen exitosas produce el mismo resultado final que
  el comportamiento actual: imagen nueva persistida, imagen anterior borrada de R2.
- **SC-003**: Ningún `Product` actualizado antes de esta corrección cambia de `image_url` ni de
  estado del objeto en R2 como consecuencia de este cambio.
- **SC-004**: Existe al menos un script de characterization que demuestra el orden de operaciones
  antes y después de la corrección, cerrando la anomalía A-44 documentada en la spec 002
  (`FR-012`).

## Out of Scope

- **Mover el borrado a un proceso asíncrono/cola post-commit** — alternativa mencionada en el
  "tratamiento acordado" de A-44, pero no elegida en esta delta; se elige la inversión síncrona del
  orden por ser la más simple y de menor riesgo.
- **Cualquier mecanismo de reintento o job de limpieza** para objetos ya huérfanos en R2 previos a
  esta corrección.
- **La subida de la imagen nueva** — ocurre fuera de este servicio, sin cambios.
- **La validación de que la URL vieja pertenece al bucket configurado** (`key_from_public_url`) —
  sin cambios, ya funciona correctamente hoy (ver Edge Cases).
- **Cualquier cambio de UI/frontend** del panel de administración — el panel ya consume el mismo
  endpoint; no hay cambio de contrato observable en el camino feliz.
- **El resto de reglas de la spec 002** (`RN-CAT-01` a `RN-CAT-11`) — no forman parte de esta delta.
- **El bloqueo de concurrencia ante dos actualizaciones simultáneas de imagen del mismo producto** —
  riesgo de código distinto, no parte de la anomalía A-44.

## Assumptions

- **Esta es una spec delta de corrección, no de característica nueva**: al igual que las specs
  002/019/020, cita nombres de función, archivo y línea explícitamente porque son el contrato
  observable que se está corrigiendo — los criterios de aceptación deben poder verificarse
  directamente contra `pos-backend` en ejecución, antes y después del cambio.
- **La autorización de negocio ya existe y no requiere una nueva ronda de entrevista**: el
  "Tratamiento acordado" de A-44 en `registro-de-anomalias.md`, clasificado ACCIDENTAL y de
  severidad Baja, satisface el Principio I de la constitución sin necesidad de testimonio adicional
  de negocio.
- **No se requiere migración de datos**: el cambio es puramente de orden de operaciones en tiempo de
  escritura (`update_product`); no hay estado existente que migrar ni reparar (ver FR-005/SC-003).
- **Se elige la inversión síncrona del orden, no el proceso asíncrono**: de las dos alternativas que
  ofrece el "tratamiento acordado" de A-44, esta delta implementa la más simple (commit primero,
  borrado después dentro de la misma petición), dejando la alternativa asíncrona fuera de alcance
  (ver Out of Scope).
- **Los nombres de función y línea citados son literales del código al momento de esta extracción**
  (2026-08-18, `pos-backend/app/api/v1/products/service.py`). Si el código cambia, esta spec debe
  re-verificarse antes de usarse como characterization test.
