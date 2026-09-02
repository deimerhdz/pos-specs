# Feature Specification: Corrección — falla al crear un tenant con usuario por migraciones rotas

**Feature Branch**: `069-fix-creacion-tenant`

**Created**: 2026-09-02

**Status**: Draft

**Input**: User description: "Hotfix: en producción y en local, crear un tenant con usuario falla
con el error de Alembic 'Can't invoke function f, as the proxy object has not yet been
established for the Alembic Operations class' (args: ck__promotions__ck_promotion_type).
Analizar y resolver."

## Resumen del defecto

Crear un tenant nuevo (con su usuario administrador) falla siempre, tanto en producción como en
local, con un error técnico interno. La causa es un defecto en dos archivos de migración de base
de datos (`alembic/versions/94b7e35f5e5e_063d_promociones_reglas_destructivo.py` y
`alembic/versions/ba4b6bd573a6_063b_promociones_retiro_estructura_.py`): ambos calculan el
nombre de una restricción de base de datos (`op.f(...)`) en el momento en que Python carga el
archivo, en vez de hacerlo dentro de la función que realmente aplica la migración. Alembic no
permite eso — solo puede resolver ese nombre cuando ya está ejecutando una migración concreta —
así que **cargar el catálogo completo de migraciones** (algo que el sistema hace cada vez que
alguien crea un tenant, para verificar que la base de datos está al día) revienta antes de
llegar siquiera a crear el registro del tenant.

Esto no es un cambio de comportamiento de negocio (Constitución, Principio II): el
comportamiento esperado — crear un tenant con su usuario administrador debe funcionar — nunca
dejó de ser el comportamiento correcto; este spec autoriza corregir el defecto que hoy lo
impide, sin alterar ninguna regla de negocio de la creación de tenants (spec 033).

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Crear un tenant nuevo con su usuario administrador (Priority: P1)

Como super-admin de la plataforma, cuando lleno el formulario de alta de un tenant nuevo (nombre,
schema, host, datos del usuario administrador, plan y ciclo de facturación) y confirmo la
creación, quiero que el tenant y su usuario administrador queden creados exitosamente, para
poder dar de alta clientes nuevos en la plataforma.

**Why this priority**: Es la única historia de este hotfix. Sin ella, ningún cliente nuevo puede
darse de alta en la plataforma — es un bloqueo total y activo en producción.

**Independent Test**: Se puede probar por completo enviando una solicitud de creación de tenant
con datos válidos (nombre, schema, host, usuario administrador, plan existente, ciclo de
facturación) y verificando que la respuesta reporta éxito y que el tenant, su schema y su
usuario administrador quedan realmente creados y consultables.

**Acceptance Scenarios**:

1. **Given** un super-admin autenticado y un plan de suscripción existente, **When** solicita
   crear un tenant nuevo con todos sus datos válidos, **Then** el tenant se crea exitosamente,
   junto con su usuario administrador, y la operación no falla por ningún error técnico interno.
2. **Given** la base de datos ya al día con todas las migraciones aplicadas, **When** el sistema
   verifica ese estado como parte de la creación de un tenant, **Then** esa verificación se
   completa sin error, usando el catálogo completo de migraciones existentes.
3. **Given** cualquier otra operación de mantenimiento que dependa de leer el catálogo completo
   de migraciones (por ejemplo, aplicar migraciones nuevas al desplegar), **When** se ejecuta,
   **Then** tampoco falla por esta misma causa.

### Edge Cases

- ¿Qué pasa con los tenants cuyo intento de creación falló mientras este defecto estuvo activo?
  Ya no quedaron creados (la operación falla antes de escribir cualquier dato, research previa a
  este spec lo confirmó) — no hay datos parciales ni inconsistentes que limpiar; solo hace falta
  volver a intentar la creación después de la corrección.
- ¿Afecta esto a tenants ya existentes, creados antes de que este defecto apareciera? No — el
  defecto solo bloquea la ruta de creación de un tenant nuevo (y, en general, cualquier operación
  que necesite cargar el catálogo completo de migraciones); los tenants existentes y sus datos no
  se ven afectados.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: El sistema DEBE permitir crear un tenant nuevo junto con su usuario administrador
  sin que la operación falle por un error técnico interno, dados datos de entrada válidos.
- **FR-002**: El sistema DEBE poder verificar que la base de datos está al día con las
  migraciones existentes (paso ya vigente antes de este defecto, spec 033) como parte de la
  creación de un tenant, sin que esa verificación por sí sola cause una falla.
- **FR-003**: El sistema DEBE poder cargar el catálogo completo de migraciones sin error en
  cualquier contexto que lo necesite (creación de tenant, y cualquier herramienta de
  mantenimiento/despliegue que dependa de lo mismo), no solo en el camino específico de este
  reporte.
- **FR-004**: La corrección NO DEBE modificar el efecto ya aplicado de ninguna migración sobre
  bases de datos donde ya se ejecutó exitosamente — el defecto está en cómo se *carga* el
  archivo de migración, no en lo que la migración le hace a los datos.
- **FR-005**: La corrección NO DEBE cambiar ninguna regla de negocio de la creación de tenants
  ya definida en spec 033 (plan obligatorio, ciclo de facturación obligatorio, validaciones de
  unicidad, etc.) — el alcance de este hotfix es exclusivamente que la operación deje de fallar
  por esta causa técnica.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: El 100% de los intentos de crear un tenant con datos válidos se completan
  exitosamente, tanto en el entorno local como en producción.
- **SC-002**: Cero intentos de creación de tenant fallan por este error técnico específico,
  verificado inmediatamente después de aplicar la corrección y de forma sostenida después.
- **SC-003**: Cualquier operación de mantenimiento/despliegue que dependa de leer el catálogo
  completo de migraciones se completa sin error, verificado al menos una vez contra el entorno
  de producción.

## Assumptions

- La causa raíz ya fue identificada antes de escribir este spec (auditoría técnica, no una
  suposición): dos archivos de migración
  (`alembic/versions/94b7e35f5e5e_063d_promociones_reglas_destructivo.py` y
  `alembic/versions/ba4b6bd573a6_063b_promociones_retiro_estructura_.py`) calculan el nombre de
  una restricción de base de datos al cargarse el archivo, en vez de dentro de la función que
  aplica la migración, que es el único momento en que Alembic permite hacerlo. Ambos archivos
  comparten el mismo patrón y deben corregirse juntos: el catálogo de migraciones no puede
  cargarse mientras cualquiera de los dos falle al importarse.
- Este defecto no es exclusivo del flujo de creación de tenants: bloquea cualquier operación que
  necesite cargar el catálogo completo de migraciones (por ejemplo, aplicar migraciones nuevas al
  desplegar). Se documenta como hallazgo relacionado, dentro del mismo alcance de corrección,
  porque comparte exactamente la misma causa raíz y la misma corrección — no se trata de una
  funcionalidad nueva ni de un alcance distinto.
- No hay datos que migrar ni limpiar: los intentos de creación de tenant que fallaron por este
  defecto no llegaron a escribir ningún dato (la falla ocurre antes de la primera escritura).
- Este hotfix no modifica ninguna regla de negocio de creación de tenants (spec 033) ni el
  manejo de errores del módulo super-admin (spec 068) — es una corrección de infraestructura de
  migraciones, acotada a los dos archivos ya identificados.
