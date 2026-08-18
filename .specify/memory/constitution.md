<!--
Sync Impact Report
===================
Versión: 2.0.0 → 3.0.0

Tipo de cambio: MAJOR. Cierra la fase de modernización (además de la fase de documentación,
ya cerrada desde la v2.0.0) y abre la fase de **EVOLUCIÓN FUNCIONAL**. Redefine de forma
incompatible el sentido de varios principios existentes y retira uno de ellos (III), lo que
por sí solo exige un bump MAJOR según la política de versionado de este mismo documento.

Principios modificados (redefinidos, no solo renombrados):
  - I. El Comportamiento Sigue Siendo Sagrado (por Defecto) → II. El Comportamiento
    Existente Sigue Protegido — REDEFINIDO. Se mantiene la exigencia de decisión de negocio
    registrada (quién/cuándo), pero ahora exige además explicitar qué comportamiento
    cambia, por qué, y qué funcionalidades se ven afectadas — y el vehículo del cambio deja
    de ser "una corrección de modernización" para ser "un spec funcional aprobado".
  - II. Los Characterization Tests son el Árbitro → III. Los Characterization Tests
    Protegen el Comportamiento Heredado — REDEFINIDO. Se mantiene el veto sobre modificar
    tests `"CONGELA comportamiento actual:"` sin autorización, pero ahora exige además que
    exista un spec del nuevo comportamiento y evidencia de que otros comportamientos
    protegidos no se ven afectados negativamente.
  - IV. Dependencias Nuevas Permitidas con Justificación → IX. Dependencias Nuevas
    Permitidas con Justificación — REDEFINIDO (ampliado). Ahora exige, además de la
    justificación en la spec, listar explícitamente alternativas consideradas e impacto
    sobre mantenimiento, seguridad y despliegue.
  - V. Ningún Cambio Retroactivo → VII. Compatibilidad con Datos Históricos — REDEFINIDO
    (ampliado). Ya no solo protege el importe de facturas emitidas: también prohíbe cambiar
    su representación histórica o aplicarles nuevas reglas de negocio con efecto
    retroactivo, y acota expresamente las reglas nuevas al ámbito temporal que defina cada
    spec.
  - VI. Todo en Español de Colombia → XIII. Todo en Español de Colombia — SIN CAMBIO DE
    FONDO, solo renumerado. No formaba parte de las reglas específicas de la fase de
    modernización, así que la enmienda no lo toca.

Principios retirados:
  - III. Estrangulamiento antes que Reescritura — retirado como principio independiente.
    Era específico de la modernización de código legado (extracción módulo por módulo). Su
    exigencia de no mezclar cambios grandes y verificar de forma aislada queda absorbida,
    para el contexto de nueva funcionalidad, por el nuevo Principio VI (Evolución
    Incremental) — que ya no habla de "extracción de módulo legado" sino de no mezclar
    nuevas funcionalidades, refactors, arquitectura, migraciones y cambios de
    comportamiento en un mismo incremento.

Principios añadidos (nuevos, formalizan las reglas de la fase de evolución funcional):
  - I. Las Nuevas Funcionalidades Nacen de un Spec
  - IV. Los Nuevos Specs Pueden Introducir Nuevo Comportamiento
  - V. Nuevas Funcionalidades Antes que Refactorizaciones Oportunistas
  - VI. Evolución Incremental
  - VIII. Evolución del Modelo de Datos
  - X. Verificación Obligatoria
  - XI. Decisiones de Negocio Frente a Decisiones Técnicas
  - XII. Trazabilidad

Secciones reescritas:
  - Alcance del Proyecto y de esta Fase → actualizada de fase de modernización a fase de
    evolución funcional (alcance, entregables y límites de esta nueva fase; los dos
    repositorios en producción no cambian).
  - Flujo de Trabajo de Modernización → renombrada y reescrita como Flujo de Trabajo de
    Evolución Funcional (spec antes que código, protección de tests de characterization,
    verificación obligatoria, trazabilidad, y el criterio de cuándo un spec se considera
    completado).

Secciones añadidas:
  - Principio Rector de esta Fase — preámbulo que contrasta el mandato de la modernización
    ("preservar el sistema existente") con el de la evolución funcional ("cambiar el
    sistema de forma intencionada, especificada y verificable"), y cierra con la regla que
    gobierna toda esta fase: la ausencia de un spec no autoriza un cambio funcional.

Secciones sin cambios estructurales: Gobernanza (actualizada en fechas, versión y en el
número/lista de principios referenciados en la revisión de cumplimiento).

Plantillas dependientes:
  - .specify/templates/plan-template.md → pendiente de revisión manual (no verificado en
    esta sesión).
  - .specify/templates/spec-template.md → pendiente de revisión manual (no verificado en
    esta sesión).
  - .specify/templates/tasks-template.md → pendiente de revisión manual (no verificado en
    esta sesión).

TODOs / seguimiento pendiente: ninguno nuevo. La fecha de ratificación original
(2026-08-15) se conserva sin cambios; esta enmienda solo actualiza la fecha de última
modificación. El documento que podría "sustituir oficialmente" a
`registro-de-anomalias.md` (mencionado como posibilidad por la propia enmienda) no existe
hoy — se sigue citando ese fichero como el libro de autorizaciones vigente hasta que un
documento sucesor se declare explícitamente.
-->

# Constitución del Sistema POS — Pedidos de Mesa por QR e Inventario

## Principio Rector de esta Fase

La fase de documentación y la fase de modernización del sistema de facturación en
producción se consideran **finalizadas**. A partir de esta enmienda, el proyecto entra en
la fase de **EVOLUCIÓN FUNCIONAL**, cuyo objetivo es incorporar nuevas capacidades mediante
especificaciones funcionales verificables, manteniendo la integridad del comportamiento
existente y la compatibilidad con las facturas ya emitidas.

Durante la modernización, el mandato era: **preservar el sistema existente**. Durante la
evolución funcional, el mandato es: **cambiar el sistema de forma intencionada,
especificada y verificable**. La modernización protegía el comportamiento existente frente
a cualquier cambio no autorizado; la evolución funcional permite cambiarlo cuando existe
una razón de negocio explícita y trazable — pero nunca sin ella.

**La ausencia de un spec no autoriza un cambio funcional.**

## Principios Fundamentales

### I. Las Nuevas Funcionalidades Nacen de un Spec
Toda nueva funcionalidad, modificación funcional o cambio de comportamiento debe comenzar
con una especificación en `specs/`. No se implementan nuevas funcionalidades directamente
sobre el código sin que exista previamente un spec aprobado que defina: problema que se
pretende resolver, comportamiento esperado, alcance, reglas de negocio, casos de uso,
criterios de aceptación, impacto sobre funcionalidades existentes, impacto sobre datos
existentes, y decisiones de compatibilidad.

**Razón**: el spec constituye el contrato funcional antes de comenzar la implementación.
Sin ese contrato no hay nada contra lo que verificar el resultado, ni nada que permita
rastrear después por qué el sistema cambió.

### II. El Comportamiento Existente Sigue Protegido
El comportamiento actual continúa considerándose válido por defecto. La finalización de la
modernización no implica libertad para modificar arbitrariamente el comportamiento
existente. Cuando una nueva funcionalidad requiera modificar un comportamiento existente,
dicho cambio debe estar explícitamente documentado en el spec correspondiente. Los cambios
que constituyan una decisión de negocio quedan registrados en
`specs/000-reconocimiento/registro-de-anomalias.md` (o el documento que oficialmente lo
sustituya), identificando como mínimo: qué comportamiento cambia, por qué debe cambiar,
quién tomó la decisión, cuándo se tomó, y qué funcionalidades se ven afectadas.

**Razón**: el negocio, no el criterio técnico de quien implementa, sigue siendo la única
autoridad para decidir que un comportamiento cambie — cerrar la modernización no traslada
esa autoridad a nadie más.

### III. Los Characterization Tests Protegen el Comportamiento Heredado
Los characterization tests continúan siendo la referencia del comportamiento existente. Los
tests con prefijo `"CONGELA comportamiento actual:"` no se modifican únicamente para hacer
pasar una nueva implementación. Si una nueva funcionalidad requiere cambiar un
comportamiento protegido, deben existir: (1) un spec que defina el nuevo comportamiento,
(2) una decisión de negocio que autorice el cambio, (3) una actualización explícita de los
tests afectados, y (4) evidencia de que el cambio no afecta negativamente a otros
comportamientos protegidos.

**Razón**: un test protegido en rojo sin una decisión que lo justifique se considera una
regresión, no una señal para "ajustar el test" — igual que en la modernización, el árbitro
no se puede sobornar editándolo hasta que esté de acuerdo con el código nuevo.

### IV. Los Nuevos Specs Pueden Introducir Nuevo Comportamiento
A diferencia de la fase de modernización, durante esta fase se permite modificar el
comportamiento funcional cuando el cambio está definido y aprobado mediante un spec. El
objetivo ya no es únicamente demostrar equivalencia con el sistema anterior.

**Razón**: el objetivo pasa a ser implementar el comportamiento definido por el nuevo
contrato funcional sin introducir regresiones no autorizadas — exigir equivalencia total
con el pasado bloquearía la razón de ser de esta fase.

### V. Nuevas Funcionalidades Antes que Refactorizaciones Oportunistas
La implementación de una nueva funcionalidad no se usa como justificación para realizar
refactorizaciones no relacionadas. Cada cambio debe poder asociarse a un spec, una tarea
derivada de ese spec, y un criterio de aceptación. Las mejoras técnicas que no sean
necesarias para la funcionalidad se tratan como trabajo independiente.

**Razón**: mezclar una refactorización no relacionada dentro de una feature oscurece qué
autorizó cada cambio y complica revertir uno sin arrastrar el otro.

### VI. Evolución Incremental
Las nuevas funcionalidades se implementan de forma incremental. Cada spec define
claramente su alcance y sus límites. Se evitan cambios masivos que mezclen nuevas
funcionalidades, refactorizaciones, cambios de arquitectura, migraciones de datos y
cambios de comportamiento en una misma unidad. Cuando una funcionalidad requiera modificar
varias áreas del sistema, se divide en unidades verificables siempre que sea posible.

**Razón**: mezclar clases de cambio distintas en un mismo incremento hace imposible aislar
qué causó qué cuando algo falla — el mismo riesgo que antes obligaba a modernizar módulo
por módulo, ahora aplicado a construir funcionalidad nueva en vez de extraer código legado.

### VII. Compatibilidad con Datos Históricos
Los datos históricos, especialmente las facturas ya emitidas y sus importes, son
inmutables. Ninguna nueva funcionalidad puede recalcular retroactivamente una factura
emitida, modificar sus importes históricos, cambiar su representación histórica, ni
aplicar nuevas reglas de negocio a operaciones ya finalizadas. Las nuevas reglas se
aplican únicamente al ámbito temporal definido por el spec.

**Razón**: una factura emitida es un hecho legal y contable consumado, no un valor que
ninguna evolución del sistema pueda "corregir" retroactivamente — la gestoría y el negocio
dependen de que el histórico de facturación sea inmutable sin importar cómo evolucione el
sistema que las generó.

### VIII. Evolución del Modelo de Datos
Las modificaciones del modelo de datos especifican explícitamente: nuevas entidades,
nuevos campos, cambios de relaciones, valores por defecto, compatibilidad con datos
existentes, estrategia de migración, y estrategia de rollback cuando sea aplicable. Las
migraciones no pueden alterar el significado histórico de los datos existentes.

**Razón**: un cambio de esquema no especificado es un cambio de comportamiento invisible
hasta que rompe algo en producción — y, sin estrategia de rollback declarada de antemano,
revertirlo bajo presión es cuando más probable es hacerlo mal.

### IX. Dependencias Nuevas Permitidas con Justificación
Las dependencias nuevas están permitidas cuando aportan valor significativo a la
funcionalidad o arquitectura. Cada nueva dependencia se justifica en el spec
correspondiente, indicando: problema que resuelve, por qué no resulta suficiente la
biblioteca estándar, alternativas consideradas, e impacto sobre mantenimiento, seguridad y
despliegue. Se mantiene la preferencia por soluciones simples y por la biblioteca estándar
de Node cuando resulte adecuada.

**Razón**: negar toda dependencia nueva a priori fuerza reinventar herramientas ya
resueltas, pero adoptarlas sin control reintroduce superficie de cambio no auditada. La
justificación explícita en la spec es el punto de control, ahora exigiendo también
alternativas consideradas e impacto, no solo el problema que resuelve.

### X. Verificación Obligatoria
Toda nueva funcionalidad se verifica mediante una combinación adecuada de: tests
existentes, characterization tests, tests de la nueva funcionalidad, criterios de
aceptación definidos en el spec, y verificación de integración cuando corresponda.

**Razón**: el resultado esperado ya no es necesariamente la equivalencia con el
comportamiento anterior — es la conformidad con el nuevo spec y la ausencia de regresiones
no autorizadas, y esa conformidad solo se demuestra verificando, no asumiendo.

### XI. Decisiones de Negocio Frente a Decisiones Técnicas
Las decisiones de negocio y las decisiones técnicas permanecen diferenciadas. Una decisión
técnica puede determinar cómo implementar una funcionalidad. Una decisión de negocio
determina qué comportamiento debe tener el sistema. Cuando exista conflicto entre el
comportamiento heredado y una nueva necesidad de negocio, el spec hace explícito el cambio
y registra la decisión correspondiente.

**Razón**: confundir ambos tipos de decisión es lo que permite que un cambio técnico se
cuele en producción como si fuera una decisión de negocio ya tomada, sin haberlo sido
nunca.

### XII. Trazabilidad
Toda nueva funcionalidad debe poder rastrearse mediante la cadena: Necesidad → Spec →
Decisión → Implementación → Tests → Verificación. El proyecto evita cambios cuyo origen o
propósito no puedan determinarse posteriormente.

**Razón**: sin esta cadena completa, ningún cambio pasado puede auditarse cuando haga
falta entender por qué el sistema se comporta como se comporta — la misma exigencia de
trazabilidad que ya regía la modernización, ahora aplicada también al "qué" funcional, no
solo al "por qué se corrigió algo".

### XIII. Todo en Español de Colombia
Toda la documentación, el registro de anomalías, los nombres de tests de
characterization, los mensajes de commit relacionados con esta evolución, los comentarios
y cualquier artefacto producido en esta fase se escriben en español de Colombia
(vocabulario y convenciones propias de Colombia, no español neutro ni de otra región).

**Razón**: es el idioma y la variedad dialectal del resto del proyecto (código, commits,
documentación previa) y del negocio y sus usuarios. Mantener la variedad específica evita
fricción de términos que en otras regiones hispanohablantes tienen sentidos distintos,
justo en los puntos donde la precisión importa más (specs, decisiones de negocio,
criterios de aceptación).

## Alcance del Proyecto y de esta Fase

El sistema cubre dos repositorios independientes, ambos **en producción**:
- **`../pos-backend`**: API (FastAPI + PostgreSQL 16, schema-per-tenant), responsable de
  catálogo, inventario, caja, ventas, facturación y el flujo de gestión de mesas por QR
  (carrito por comensal → consolidación → cobro → facturación).
- **`../pos-heladeria`**: frontend (Angular), consumidor de esa API. Referido
  coloquialmente por el negocio como "pos-frontend"; su nombre real en disco y en el
  repositorio remoto es `pos-heladeria`.

La fase de documentación (`specs/000-reconocimiento/`) y la fase de modernización están
**cerradas**. El entregable principal de la primera, el registro de anomalías
(`specs/000-reconocimiento/registro-de-anomalias.md`), no se archiva: sigue siendo el
libro de autorizaciones vivo que el Principio II exige consultar y alimentar cada vez que
una nueva funcionalidad cambie un comportamiento existente.

A partir de esta enmienda, el **objetivo del proyecto es evolucionar funcionalmente**
ambos repositorios mediante specs verificables (Principio I), preservando por defecto el
comportamiento existente salvo decisión explícita de negocio (Principio II) y la
inmutabilidad de los datos históricos (Principio VII). Son artefactos de salida de esta
fase, por funcionalidad:
1. Una spec funcional en `specs/` que defina problema, comportamiento esperado, alcance,
   reglas de negocio, casos de uso, criterios de aceptación, impacto sobre funcionalidades
   y datos existentes, y decisiones de compatibilidad.
2. Cualquier decisión de negocio que autorice un cambio de comportamiento existente,
   registrada en `specs/000-reconocimiento/registro-de-anomalias.md` con quién y cuándo.
3. Los characterization tests afectados, actualizados explícitamente cuando el spec lo
   autorice — nunca en silencio.
4. La implementación del comportamiento definido por el spec.
5. Los tests de la nueva funcionalidad y la verificación de que no introduce regresiones
   no autorizadas.

Están **fuera de alcance** en esta fase: implementar cualquier cambio funcional sin un
spec que lo respalde (Principio I), cambiar comportamiento existente sin decisión de
negocio registrada (Principio II), mezclar refactorizaciones no relacionadas dentro de una
funcionalidad (Principio V), cambios masivos que combinen varias clases de cambio a la vez
(Principio VI), y cualquier alteración — directa o indirecta, retroactiva o de
representación — de facturas ya emitidas (Principio VII).

## Flujo de Trabajo de Evolución Funcional

- Antes de tocar código, toda nueva funcionalidad o cambio de comportamiento tiene un spec
  aprobado en `specs/` que define su contrato funcional completo (Principio I).
- Si el spec requiere cambiar un comportamiento existente, esa decisión de negocio queda
  registrada en `specs/000-reconocimiento/registro-de-anomalias.md` con quién, cuándo, qué
  cambia, por qué, y qué funcionalidades se ven afectadas, **antes** de implementar el
  cambio (Principio II).
- Los tests `"CONGELA comportamiento actual:"` no se tocan salvo que el spec lo autorice
  explícitamente; modificarlos siempre cita, en el mismo commit, la decisión que lo
  permite, junto con evidencia de que ningún otro comportamiento protegido se ve afectado
  (Principio III).
- Un spec puede definir comportamiento nuevo, no solo preservar el existente — el criterio
  de éxito es la conformidad con ese spec y la ausencia de regresiones no autorizadas, no
  la equivalencia total con el sistema anterior (Principio IV).
- Ninguna refactorización no relacionada con la funcionalidad en curso viaja en el mismo
  cambio; se trata como trabajo independiente, con su propio spec si aplica (Principio V).
- Cada funcionalidad se entrega en unidades verificables, sin mezclar nueva funcionalidad,
  refactorización, cambio de arquitectura, migración de datos y cambio de comportamiento
  en un mismo incremento (Principio VI).
- Ningún cambio recalcula, reemite ni altera una factura ya emitida, su importe, ni su
  representación histórica, sin excepción (Principio VII).
- Toda modificación del modelo de datos especifica entidades/campos nuevos, valores por
  defecto, compatibilidad con datos existentes, estrategia de migración y de rollback
  antes de ejecutarse (Principio VIII).
- Cualquier dependencia nueva se justifica en la spec correspondiente —problema que
  resuelve, alternativas consideradas, impacto en mantenimiento/seguridad/despliegue— y se
  aprueba antes de añadirse, con preferencia por la biblioteca estándar cuando resuelva lo
  mismo (Principio IX).
- Toda funcionalidad se verifica con la combinación de tests existentes, characterization
  tests, tests propios de la funcionalidad y los criterios de aceptación del spec, antes
  de darse por completa (Principio X).
- Cuando el comportamiento heredado entra en conflicto con una necesidad de negocio nueva,
  el spec lo hace explícito y registra cuál de las dos decisiones prevalece y por qué
  (Principio XI).
- Toda spec, registro de anomalías, nombre de test de characterization y mensaje de commit
  relacionado con esta fase se escribe en español de Colombia (Principio XIII).

**Un spec se considera completado cuando**: el comportamiento esperado está definido; los
criterios de aceptación están satisfechos; la implementación está terminada; los tests
correspondientes pasan; los characterization tests siguen pasando salvo cambios
explícitamente autorizados; las migraciones, si existen, han sido verificadas; y no quedan
cambios funcionales fuera del alcance aprobado. Cada uno de estos puntos debe poder
rastrearse hacia atrás siguiendo la cadena Necesidad → Spec → Decisión → Implementación →
Tests → Verificación (Principio XII).

## Gobernanza

Esta constitución prevalece sobre cualquier otra guía, plantilla o convención de este
repositorio de specs (`pos-specs`) en caso de conflicto. No rige el código fuente de
`pos-backend` ni `pos-heladeria` en sí mismo — rige cómo se evoluciona y protege ese
código durante esta fase.

**Enmiendas**: cualquier cambio a esta constitución (incluyendo alterar, debilitar o
eliminar alguno de sus trece principios) requiere una decisión explícita y por escrito,
registrada en el propio commit o PR que modifica este fichero, con la razón del cambio.

**Versionado**: este documento usa versionado semántico (`MAJOR.MINOR.PATCH`):
- **MAJOR**: eliminación o redefinición incompatible de un principio existente.
- **MINOR**: adición de un nuevo principio o sección, o ampliación material de una guía
  existente.
- **PATCH**: aclaraciones de redacción, correcciones tipográficas o refinamientos que no
  cambian el sentido normativo del texto.

**Revisión de cumplimiento**: cualquier trabajo producido bajo esta fase (specs
funcionales, decisiones de negocio, characterization tests, implementaciones) debe poder
verificarse contra los trece principios antes de considerarse completo. En particular:
toda funcionalidad nueva tiene un spec previo (Principio I); todo cambio de comportamiento
existente tiene una decisión de negocio citada con quién y cuándo (Principio II); ningún
test `"CONGELA comportamiento actual:"` en rojo queda sin decisión que lo ampare
(Principio III); ninguna refactorización no relacionada viaja junto a una funcionalidad
(Principio V); ningún incremento mezcla varias clases de cambio a la vez (Principio VI);
ninguna factura ya emitida fue alterada en importe o representación (Principio VII);
ninguna migración de datos carece de estrategia de compatibilidad y rollback
(Principio VIII); toda dependencia nueva está justificada y aprobada (Principio IX);
ninguna funcionalidad se da por completa sin verificación (Principio X); y toda la cadena
Necesidad → Spec → Decisión → Implementación → Tests → Verificación es rastreable
(Principio XII).

**Version**: 3.0.0 | **Ratified**: 2026-08-15 | **Last Amended**: 2026-08-18
