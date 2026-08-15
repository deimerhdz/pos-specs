<!--
Sync Impact Report
===================
Versión: (plantilla sin ratificar) → 1.0.0
Tipo de cambio: MINOR→MAJOR tratado como ratificación inicial (1.0.0), primera versión formal del documento.

Principios definidos (nuevos, 5 de 5 según instrucción explícita del usuario):
  1. El Comportamiento Actual es Sagrado
  2. Evidencia Obligatoria
  3. Los Hallazgos se Registran, no se Corrigen
  4. Cero Dependencias Nuevas y Cero Refactors
  5. Todo en Español

Secciones añadidas:
  - Alcance del Proyecto y de esta Fase
  - Flujo de Trabajo de Documentación y Evidencia
  - Gobernanza

Secciones eliminadas: ninguna (documento inicial).

Plantillas dependientes:
  - .specify/templates/plan-template.md → pendiente de revisión manual (no verificado en esta sesión)
  - .specify/templates/spec-template.md → pendiente de revisión manual (no verificado en esta sesión)
  - .specify/templates/tasks-template.md → pendiente de revisión manual (no verificado en esta sesión)

TODOs / seguimiento pendiente:
  - TODO(RATIFICATION_DATE): se usa 2026-08-15 (fecha de esta sesión) como fecha de ratificación
    porque no se encontró una versión previa firmada del documento ni una fecha de adopción
    anterior declarada por el negocio. Si existe una fecha de adopción anterior real, corregir.
  - Discrepancia de nombres resuelta por decisión explícita del usuario en esta sesión
    (2026-08-15): el repositorio que el negocio llama coloquialmente "pos-frontend" es, en
    disco y en el remoto git, `pos-heladeria` (Angular, `package.json` → `"name": "frontend"`,
    `git remote` → `github.com/deimerhdz/pos-heladeria.git`). Este documento usa el nombre
    verificado `pos-heladeria`.
-->

# Constitución del Sistema POS — Pedidos de Mesa por QR e Inventario

## Principios Fundamentales

### I. El Comportamiento Actual es Sagrado
El comportamiento observable actual del sistema (backend `pos-backend` y frontend
`pos-heladeria`, incluyendo API, cálculos, flujos de mesas por QR, inventario, caja y
facturación) es la línea base autorizada. Ningún cambio de comportamiento observable se
introduce sin una decisión explícita y por escrito del negocio, **incluso si el
comportamiento parece un error, un bug o una inconsistencia**. Documentar no es autorizar:
describir un comportamiento en esta fase no habilita, por sí mismo, a corregirlo.

**Razón**: el sistema aún no factura dinero real, pero terceros (la gestoría) dependen de
sus resultados exactos, peso a peso (COP), para su propio trabajo. Un cambio de
comportamiento no autorizado —aunque "mejore" el sistema— puede romper silenciosamente esa
dependencia externa. La única autoridad para decidir que un comportamiento cambie es el
negocio, nunca el criterio técnico de quien documenta.

### II. Evidencia Obligatoria
Toda afirmación sobre lo que el sistema hace debe ir acompañada de su evidencia, que debe
ser una de las siguientes:
- **Código**: fichero y líneas exactas (ej. `app/services/mesas.py:120-134`).
- **Dato real**: un valor observado directamente en la base de datos u otro almacén de
  datos del sistema, con la consulta o el método usado para obtenerlo.
- **Testimonio del negocio**: una afirmación de una persona del negocio, con **nombre y
  fecha**, registrada textualmente o parafraseada sin alterar su sentido.

Cualquier afirmación que no tenga una de estas tres evidencias **no es un hecho**: se marca
explícitamente como `SUPOSICIÓN` en el propio texto y se convierte en una pregunta abierta
dirigida al negocio, nunca se presenta como comportamiento confirmado.

**Razón**: la salida de este trabajo sirve como base para decisiones que afectan a un
tercero (la gestoría) y a dinero real, aunque hoy no se facture en vivo. Una afirmación sin
evidencia verificable contamina esa base y es indistinguible de una opinión.

### III. Los Hallazgos no se Corrigen, se Registran
Todo bug, riesgo o comportamiento extraño ("rareza") descubierto durante la documentación se
anota en el registro de anomalías con su clasificación (severidad, área afectada, evidencia
según el Principio II). Corregirlo **no** es una acción de esta fase: es una decisión que
corresponde a un módulo posterior, explícitamente fuera del alcance de este trabajo.

**Razón**: mezclar "documentar" con "arreglar sobre la marcha" es la forma más rápida de
romper el Principio I sin darse cuenta. Separar el registro de la corrección obliga a que
cada corrección futura pase por una decisión consciente del negocio, con el hallazgo ya
documentado como insumo.

### IV. Cero Dependencias Nuevas y Cero Refactors
En esta fase no se añade ningún paquete o librería nueva a `pos-backend` ni a
`pos-heladeria`, y no se reorganiza, renombra ni reestructura código existente. Los tests de
caracterización que se escriban para capturar el comportamiento actual usan
**exclusivamente** el framework nativo de Python (`unittest`) — sin `pytest`, sin librerías
de mocking de terceros, sin fixtures externos.

**Razón**: cualquier dependencia nueva o refactor es, en sí mismo, un cambio de superficie
del sistema que puede alterar comportamiento observable (aunque sea de forma indirecta, vía
versiones, side effects de import, o reordenamiento de código), lo cual viola el Principio I.
Restringir los tests de caracterización a `unittest` evita introducir esa superficie de
cambio incluso en el propio trabajo de verificación.

### V. Todo en Español
Toda la documentación, el registro de anomalías, los nombres de tests de caracterización,
los comentarios y cualquier artefacto producido en esta fase se escriben en español.

**Razón**: es el idioma del resto del proyecto (código, commits, README de `pos-backend`) y
el idioma del negocio y sus usuarios. Escribir en otro idioma introduce fricción de
traducción exactamente en el punto donde la precisión importa más (evidencia y hallazgos).

## Alcance del Proyecto y de esta Fase

El sistema cubre dos repositorios independientes:
- **`../pos-backend`**: API (FastAPI + PostgreSQL 16, schema-per-tenant), responsable de
  catálogo, inventario, caja, ventas, facturación y el flujo de gestión de mesas por QR
  (carrito por comensal → consolidación → cobro → facturación).
- **`../pos-heladeria`**: frontend (Angular), consumidor de esa API. Referido
  coloquialmente por el negocio como "pos-frontend"; su nombre real en disco y en el
  repositorio remoto es `pos-heladeria`.

El **objetivo de esta fase es exclusivamente documentar y proteger** el comportamiento
actual de ambos repositorios — no mejorarlo, no optimizarlo, no corregirlo. Son artefactos
de salida de esta fase:
1. Documentación del comportamiento observado, cada afirmación con su evidencia
   (Principio II).
2. El registro de anomalías (Principio III).
3. Tests de caracterización en `unittest` que capturan el comportamiento actual tal como es,
   no como debería ser.

Están **fuera de alcance** en esta fase: cambios funcionales, corrección de bugs,
refactors, actualización de dependencias, cambios de arquitectura, y cualquier otra acción
que modifique comportamiento observable o la estructura del código de `pos-backend` o
`pos-heladeria`.

## Flujo de Trabajo de Documentación y Evidencia

- Antes de documentar un flujo o cálculo, se identifica el código fuente exacto que lo
  implementa (fichero y líneas) antes de describir su comportamiento en prosa.
- Cuando el código no basta para confirmar el comportamiento (por ejemplo, dependencias de
  configuración, datos de entorno, o comportamiento en producción), se contrasta con datos
  reales de la base de datos o con testimonio del negocio, nunca se infiere en silencio.
- Cada hallazgo que entra al registro de anomalías incluye, como mínimo: descripción,
  evidencia (Principio II), clasificación de severidad/tipo, y área o módulo afectado.
  No incluye una propuesta de corrección como parte de esta fase.
- Los tests de caracterización se nombran y documentan de forma que dejen explícito que
  capturan comportamiento *actual observado*, no comportamiento *deseado* — su fallo ante un
  cambio futuro es la señal de que ese cambio requiere la decisión explícita del Principio I.
- Toda `SUPOSICIÓN` señalada en el texto se acompaña de la pregunta concreta que, una vez
  respondida por el negocio, la convierte en evidencia.

## Gobernanza

Esta constitución prevalece sobre cualquier otra guía, plantilla o convención de este
repositorio de specs (`pos-specs`) en caso de conflicto. No rige el código fuente de
`pos-backend` ni `pos-heladeria` en sí mismo — rige cómo se documenta y protege ese código
durante esta fase.

**Enmiendas**: cualquier cambio a esta constitución (incluyendo alterar, debilitar o
eliminar alguno de los cinco principios) requiere una decisión explícita y por escrito,
registrada en el propio commit o PR que modifica este fichero, con la razón del cambio.

**Versionado**: este documento usa versionado semántico (`MAJOR.MINOR.PATCH`):
- **MAJOR**: eliminación o redefinición incompatible de un principio existente.
- **MINOR**: adición de un nuevo principio o sección, o ampliación material de una guía
  existente.
- **PATCH**: aclaraciones de redacción, correcciones tipográficas o refinamientos que no
  cambian el sentido normativo del texto.

**Revisión de cumplimiento**: cualquier trabajo producido bajo esta fase (documentación,
registro de anomalías, tests de caracterización) debe poder verificarse contra los cinco
principios antes de considerarse completo. En particular: toda afirmación tiene evidencia
trazable (Principio II), ningún hallazgo fue corregido en el propio código de producción
(Principio III), y no se introdujo ninguna dependencia nueva ni refactor (Principio IV).

**Version**: 1.0.0 | **Ratified**: 2026-08-15 | **Last Amended**: 2026-08-15
