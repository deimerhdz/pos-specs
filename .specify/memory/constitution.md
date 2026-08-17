<!--
Sync Impact Report
===================
Versión: 1.0.0 → 2.0.0

Tipo de cambio: MAJOR. Cierra la fase de documentación y abre la fase de modernización.
Se elimina un principio existente (III) y se invierte el sentido normativo de otro (IV),
ambos cambios incompatibles con la versión anterior.

Principios modificados:
  - I. El Comportamiento Actual es Sagrado → I. El Comportamiento Sigue Siendo Sagrado
    (por Defecto) — REDEFINIDO. Antes el comportamiento no podía cambiar bajo ninguna
    circunstancia durante la fase de documentación. Ahora sí puede cambiar, pero solo
    mediante una decisión de negocio registrada explícitamente (documento, quién, cuándo).
  - IV. Cero Dependencias Nuevas y Cero Refactors → IV. Dependencias Nuevas Permitidas
    con Justificación — REDEFINIDO (inversión). Antes prohibidas sin excepción; ahora
    permitidas con justificación en la spec y aprobación explícita.
  - V. Todo en Español → VI. Todo en Español de Colombia — REDEFINIDO (afinado a la
    variedad dialectal colombiana, no español genérico).

Principios eliminados:
  - II. Evidencia Obligatoria — retirado como principio independiente. Era específico de
    la fase de documentación (evidenciar afirmaciones sobre comportamiento observado). Su
    exigencia de trazabilidad queda absorbida por el nuevo Principio I (citar quién y
    cuándo tomó la decisión) y el nuevo Principio II (citar la decisión en el commit).
  - III. Los Hallazgos no se Corrigen, se Registran — retirado. Contradice el propósito
    mismo de la fase de modernización, cuyo objetivo es corregir/modernizar bajo control,
    no solo registrar sin tocar. El registro de anomalías (specs/000-reconocimiento/
    registro-de-anomalias.md) permanece como el libro de autorizaciones que el nuevo
    Principio I exige consultar y alimentar.

Principios añadidos (nuevos, formalizan prácticas de la fase de modernización):
  - II. Los Characterization Tests son el Árbitro
  - III. Estrangulamiento antes que Reescritura
  - V. Ningún Cambio Retroactivo

Secciones reescritas:
  - Alcance del Proyecto y de esta Fase → actualizada de fase de documentación a fase de
    modernización (alcance, entregables y límites de esta nueva fase).
  - Flujo de Trabajo de Documentación y Evidencia → renombrada y reescrita como Flujo de
    Trabajo de Modernización (estrangulamiento por módulo, characterization tests, golden
    master, registro de decisiones).

Secciones sin cambios estructurales: Gobernanza (actualizada solo en fechas, versión y
número de principios referenciados).

Plantillas dependientes:
  - .specify/templates/plan-template.md → pendiente de revisión manual (no verificado en
    esta sesión).
  - .specify/templates/spec-template.md → pendiente de revisión manual (no verificado en
    esta sesión).
  - .specify/templates/tasks-template.md → pendiente de revisión manual (no verificado en
    esta sesión).

TODOs / seguimiento pendiente: ninguno nuevo. La fecha de ratificación original (2026-08-15)
se conserva sin cambios; esta enmienda solo actualiza la fecha de última modificación.
-->

# Constitución del Sistema POS — Pedidos de Mesa por QR e Inventario

## Principios Fundamentales

### I. El Comportamiento Sigue Siendo Sagrado (por Defecto)
El comportamiento observable actual del sistema (backend `pos-backend` y frontend
`pos-heladeria`, incluyendo API, cálculos, flujos de mesas por QR, inventario, caja y
facturación) sigue siendo la línea base por defecto. A diferencia de la fase de
documentación ya cerrada, ahora **sí** puede cambiar — pero únicamente mediante una
decisión de negocio registrada explícitamente en
`specs/000-reconocimiento/registro-de-anomalias.md` (o el documento que lo sustituya),
citando **quién** tomó la decisión y **cuándo**. Ningún cambio de comportamiento se
introduce por criterio técnico, "mejora" percibida o limpieza de código sin que esa
decisión exista primero, por escrito, en el registro.

**Razón**: la modernización existe para corregir y modernizar, no para congelar el
sistema indefinidamente — pero el sistema está en producción y terceros (la gestoría)
dependen de sus resultados exactos. Permitir el cambio sin exigir su trazabilidad
reintroduciría el mismo riesgo que la fase de documentación existía para evitar. La
única autoridad para decidir que un comportamiento cambie sigue siendo el negocio, nunca
el criterio técnico de quien modifica el código.

### II. Los Characterization Tests son el Árbitro
Los characterization tests son el árbitro final de si la modernización preserva el
comportamiento acordado. Modificar cualquier test cuyo nombre lleve el prefijo
`"CONGELA comportamiento actual:"`, o regenerar el golden master que ese test verifica,
exige citar **en el propio commit** la decisión del registro de anomalías que lo
autoriza. Un test de ese tipo en rojo, sin una decisión que lo ampare, significa que la
modernización rompió algo: **se revierte el cambio, no se "ajusta el test"** para que
vuelva a pasar.

**Razón**: sin esta regla, el árbitro se puede sobornar — basta con editar el test hasta
que esté de acuerdo con el código nuevo. Exigir la cita de la decisión en el commit hace
que cualquier cambio de comportamiento sea trazable desde el test hasta la autorización
de negocio que lo permitió, y que revertir sea siempre la opción por defecto ante la
duda.

### III. Estrangulamiento antes que Reescritura
El sistema se moderniza módulo por módulo mediante el patrón de estrangulamiento
(strangler fig), nunca por reescritura total. Cada módulo modernizado requiere, en este
orden: su propia spec, su extracción del código legado, y su verificación de
equivalencia frente al comportamiento congelado (Principio II). **Está prohibido
reescribir más de un módulo a la vez.**

**Razón**: reescribir varios módulos en paralelo, o el sistema entero de una vez, hace
imposible aislar qué cambio rompió qué comportamiento cuando algo falla — y en un
sistema en producción, con dinero real de por medio, esa trazabilidad no es negociable.
Un módulo a la vez mantiene cada extracción verificable de forma independiente.

### IV. Dependencias Nuevas Permitidas con Justificación
Las dependencias nuevas dejan de estar prohibidas. Añadir un paquete o librería nueva a
`pos-backend`, `pos-heladeria`, o a cualquier módulo extraído durante la modernización,
requiere justificación explícita en la spec del módulo correspondiente y aprobación
explícita antes de incorporarla. Cuando exista una solución equivalente en la biblioteca
estándar de Node, esta se prefiere sobre una dependencia externa.

**Razón**: la prohibición total de dependencias tenía sentido mientras el único objetivo
era documentar sin tocar nada. En modernización, negar toda dependencia nueva a priori
fuerza reinventar herramientas ya resueltas; pero adoptarlas sin control reintroduce el
mismo riesgo de superficie de cambio no auditada que la fase anterior evitaba. La
justificación en la spec y la aprobación explícita son el punto de control.

### V. Ningún Cambio Retroactivo
Las facturas ya emitidas y sus importes son intocables, sin excepción, pase lo que pase
con la lógica de facturación durante la modernización. Ningún cambio de comportamiento,
corrección de bug, ni regeneración de golden master autoriza a recalcular, reemitir o
alterar una factura ya emitida ni el importe que en su momento se calculó para ella.

**Razón**: una factura emitida es un hecho legal y contable consumado, no un valor que
la modernización pueda "corregir" retroactivamente. La gestoría y el negocio dependen de
que el histórico de facturación sea inmutable independientemente de cómo evolucione el
código que las generó.

### VI. Todo en Español de Colombia
Toda la documentación, el registro de anomalías, los nombres de tests de
characterization, los mensajes de commit relacionados con esta modernización, los
comentarios y cualquier artefacto producido en esta fase se escriben en español de
Colombia (vocabulario y convenciones propias de Colombia, no español neutro ni de otra
región).

**Razón**: es el idioma y la variedad dialectal del resto del proyecto (código, commits,
documentación previa) y del negocio y sus usuarios. Mantener la variedad específica, no
solo "español" en general, evita fricción de términos que en otras regiones
hispanohablantes tienen sentidos distintos, justo en los puntos donde la precisión
importa más (specs, decisiones de negocio, hallazgos).

## Alcance del Proyecto y de esta Fase

El sistema cubre dos repositorios independientes, ambos **en producción**:
- **`../pos-backend`**: API (FastAPI + PostgreSQL 16, schema-per-tenant), responsable de
  catálogo, inventario, caja, ventas, facturación y el flujo de gestión de mesas por QR
  (carrito por comensal → consolidación → cobro → facturación).
- **`../pos-heladeria`**: frontend (Angular), consumidor de esa API. Referido
  coloquialmente por el negocio como "pos-frontend"; su nombre real en disco y en el
  repositorio remoto es `pos-heladeria`.

La fase de documentación (reconocimiento del sistema, `specs/000-reconocimiento/`) está
**cerrada**. Su entregable principal, el registro de anomalías
(`specs/000-reconocimiento/registro-de-anomalias.md`), no se archiva: pasa a ser el libro
de autorizaciones vivo que el Principio I exige citar en cada cambio de comportamiento, y
sigue alimentándose con cada hallazgo nuevo que aparezca durante la modernización.

A partir de esta enmienda, el **objetivo del proyecto es modernizar** ambos repositorios,
módulo por módulo (Principio III), preservando el comportamiento acordado salvo decisión
explícita de negocio (Principio I). Son artefactos de salida de esta fase, por módulo:
1. Una spec del módulo a modernizar.
2. Sus characterization tests (`"CONGELA comportamiento actual:"`) y, cuando aplique, su
   golden master, escritos **antes** de tocar el código legado del módulo.
3. La extracción/reescritura del módulo, verificada frente a esos tests.
4. Cualquier decisión de negocio que autorice un cambio de comportamiento, registrada en
   `specs/000-reconocimiento/registro-de-anomalias.md` con quién y cuándo.

Están **fuera de alcance** en esta fase: reescribir más de un módulo a la vez
(Principio III), cambiar comportamiento sin decisión de negocio registrada
(Principio I), y cualquier alteración — directa o indirecta — de facturas ya emitidas
(Principio V).

## Flujo de Trabajo de Modernización

- Antes de tocar el código legado de un módulo, se escriben sus characterization tests
  (`"CONGELA comportamiento actual:"`) y, si aplica, se genera su golden master, de forma
  que capturen el comportamiento *actual observado*, no el *deseado*.
- La extracción o reescritura del módulo se hace contra esos tests en verde. Un test
  `"CONGELA comportamiento actual:"` en rojo durante o después de la extracción se trata
  como una regresión: se revierte el cambio que lo rompió, salvo que exista ya una
  decisión de negocio en el registro de anomalías que autorice exactamente ese cambio de
  comportamiento (Principios I y II).
- Modificar un test `"CONGELA comportamiento actual:"` o regenerar un golden master
  siempre cita, en el mismo commit, la decisión del registro de anomalías que lo
  autoriza. Sin esa cita, el cambio no se hace.
- Ninguna extracción de módulo empieza si otro módulo sigue en proceso de reescritura
  (Principio III): se completa y verifica uno antes de empezar el siguiente.
- Cualquier dependencia nueva que un módulo necesite se justifica en la spec de ese
  módulo, con preferencia explícita por la biblioteca estándar de Node cuando resuelva el
  mismo problema, y se aprueba antes de añadirse (Principio IV).
- Ninguna tarea de modernización toca facturas ya emitidas ni sus importes, sin
  excepción (Principio V).
- Toda spec, registro de anomalías, nombre de test de characterization y mensaje de
  commit relacionado con la modernización se escribe en español de Colombia
  (Principio VI).

## Gobernanza

Esta constitución prevalece sobre cualquier otra guía, plantilla o convención de este
repositorio de specs (`pos-specs`) en caso de conflicto. No rige el código fuente de
`pos-backend` ni `pos-heladeria` en sí mismo — rige cómo se moderniza y protege ese
código durante esta fase.

**Enmiendas**: cualquier cambio a esta constitución (incluyendo alterar, debilitar o
eliminar alguno de los seis principios) requiere una decisión explícita y por escrito,
registrada en el propio commit o PR que modifica este fichero, con la razón del cambio.

**Versionado**: este documento usa versionado semántico (`MAJOR.MINOR.PATCH`):
- **MAJOR**: eliminación o redefinición incompatible de un principio existente.
- **MINOR**: adición de un nuevo principio o sección, o ampliación material de una guía
  existente.
- **PATCH**: aclaraciones de redacción, correcciones tipográficas o refinamientos que no
  cambian el sentido normativo del texto.

**Revisión de cumplimiento**: cualquier trabajo producido bajo esta fase (specs de
módulo, characterization tests, extracciones, registro de anomalías) debe poder
verificarse contra los seis principios antes de considerarse completo. En particular:
todo cambio de comportamiento tiene una decisión de negocio citada con quién y cuándo
(Principio I), ningún test `"CONGELA comportamiento actual:"` en rojo queda sin decisión
que lo ampare (Principio II), ningún momento tiene más de un módulo en reescritura
simultánea (Principio III), toda dependencia nueva está justificada y aprobada
(Principio IV), y ninguna factura ya emitida fue alterada (Principio V).

**Version**: 2.0.0 | **Ratified**: 2026-08-15 | **Last Amended**: 2026-08-17
