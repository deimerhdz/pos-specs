# Feature Specification: Corrección — falla al crear un tenant por una referencia entre schemas no resuelta

**Feature Branch**: `070-fix-metadata-tenant-fk`

**Created**: 2026-09-02

**Status**: Draft

**Input**: User description: "Hotfix: al crear un tenant con usuario, después de corregir el
NameError de Alembic (spec 069), la creación sigue fallando con
sqlalchemy.exc.NoReferencedTableError: Foreign key associated with column
'payment_methods.catalog_id' could not find table 'shared.payment_method_catalog'. La causa es
que app/core/db.py::get_tenant_specific_metadata() arma una copia de metadata aislada solo con
las tablas del tenant, y no incluye la tabla compartida que una de ellas referencia por llave
foránea. Analizar y resolver mediante hotfix, sin tocar nada fuera de esta causa."

## Resumen del defecto

Crear un tenant nuevo sigue fallando **incluso después** de aplicar la corrección de spec 069
(que resolvió el error de Alembic que bloqueaba el mismo flujo un paso antes). Ahora falla en el
paso siguiente: al crear las tablas propias del tenant nuevo, el sistema no logra generar
correctamente la definición de una de ellas (`payment_methods`), porque esa tabla hace
referencia a una tabla compartida entre todos los tenants (el catálogo de métodos de pago de la
plataforma, ya existente) y, al preparar internamente solo la definición de las tablas del
tenant, esa referencia queda sin resolver.

Este defecto es preexistente e independiente del corregido en spec 069 — comparte el mismo
síntoma visible para quien usa el sistema (crear un tenant falla) pero tiene una causa técnica
completamente distinta, en un paso posterior del mismo proceso. No es un cambio de comportamiento
de negocio (Constitución, Principio II): crear un tenant con su usuario administrador, incluyendo
que ese tenant pueda usar el catálogo de métodos de pago de la plataforma, siempre fue el
comportamiento esperado (spec 032); este spec autoriza corregir el defecto técnico que hoy lo
impide.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Crear un tenant nuevo con su usuario administrador, hasta el final (Priority: P1)

Como super-admin de la plataforma, cuando creo un tenant nuevo, quiero que el proceso se
complete de principio a fin — el tenant, su schema de datos propio (con todas sus tablas, incluidas
las que dependen del catálogo compartido de la plataforma) y su usuario administrador — para
poder dar de alta clientes nuevos sin que la operación se interrumpa a mitad de camino.

**Why this priority**: Es la continuación directa del incidente de spec 069: corregir el primer
bloqueo no sirve de nada en la práctica si el proceso vuelve a fallar en el siguiente paso. Sin
esta corrección, la plataforma sigue sin poder dar de alta clientes nuevos.

**Independent Test**: Se puede probar por completo creando un tenant con datos válidos y
verificando que la operación reporta éxito, que todas las tablas del schema del tenant nuevo
quedan creadas (incluida la de métodos de pago), y que ese tenant puede activar métodos de pago
del catálogo compartido de la plataforma sin ningún error adicional.

**Acceptance Scenarios**:

1. **Given** un super-admin autenticado, un plan existente, y el catálogo de métodos de pago de
   la plataforma ya poblado, **When** solicita crear un tenant nuevo, **Then** la operación se
   completa exitosamente y todas las tablas del schema del tenant nuevo quedan creadas — sin
   ningún error relacionado con el catálogo de métodos de pago ni con ninguna otra tabla
   compartida que alguna tabla del tenant referencie.
2. **Given** un tenant recién creado por el escenario anterior, **When** un usuario de ese tenant
   consulta o activa un método de pago del catálogo de la plataforma, **Then** la operación
   funciona con normalidad — confirma que la tabla y su referencia al catálogo quedaron bien
   formadas, no solo que la creación "no falló".

### Edge Cases

- ¿Qué pasa con los intentos de creación de tenant que fallaron por este defecto mientras estuvo
  activo? Igual que en spec 069: la operación falla antes de dejar datos parciales
  inconsistentes (la transacción se revierte por completo) — no hay nada que limpiar, solo
  reintentar después de la corrección.
- ¿Afecta esto a tenants ya existentes y a su catálogo de métodos de pago ya activados? No — el
  defecto solo bloquea la creación de un tenant **nuevo**; los tenants existentes, sus datos, y
  su relación ya establecida con el catálogo compartido no se ven afectados.
- ¿Hay otras tablas de tenant con el mismo tipo de referencia a una tabla compartida, además de
  la de métodos de pago? Se audita como parte de este spec (ver Assumptions) — la corrección debe
  cubrir el caso general, no solo el caso puntual reportado, para no dejar el mismo defecto
  esperando a aparecer con la próxima tabla que dependa de algo compartido.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: El sistema DEBE completar la creación de un tenant nuevo de principio a fin,
  incluyendo la creación de todas las tablas de su schema propio, sin fallar por no poder
  resolver una referencia hacia una tabla compartida entre tenants.
- **FR-002**: El sistema DEBE seguir permitiendo que las tablas de un tenant referencien
  correctamente datos del catálogo compartido de la plataforma (por ejemplo, el catálogo de
  métodos de pago, spec 032) — la corrección no puede romper esa relación para hacer que la
  creación del tenant "pase".
- **FR-003**: La corrección DEBE cubrir cualquier tabla de tenant que haga este mismo tipo de
  referencia hacia una tabla compartida, no únicamente el caso puntual de métodos de pago
  reportado — evitando que el mismo defecto reaparezca la próxima vez que exista una referencia
  de este tipo.
- **FR-004**: La corrección NO DEBE alterar el esquema de ningún tenant ya existente, ni el
  contenido del catálogo compartido de la plataforma — el defecto está en cómo el sistema prepara
  internamente la definición de las tablas al crear un tenant **nuevo**, no en los datos ya
  existentes.
- **FR-005**: La corrección NO DEBE cambiar ninguna regla de negocio de la creación de tenants
  (spec 033) ni del catálogo de métodos de pago (spec 032) — el alcance de este hotfix es
  exclusivamente que la operación deje de fallar por esta causa técnica.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: El 100% de los intentos de crear un tenant con datos válidos se completan
  exitosamente de principio a fin (tenant, schema completo, usuario administrador), tanto en
  local como en producción.
- **SC-002**: Un tenant recién creado puede usar el catálogo compartido de métodos de pago de la
  plataforma sin ningún error, verificado inmediatamente después de la corrección.
- **SC-003**: Cero intentos de creación de tenant fallan por esta causa técnica específica,
  verificado de forma sostenida después de aplicar la corrección.

## Assumptions

- La causa raíz ya fue identificada antes de escribir este spec (auditoría técnica, no una
  suposición): `app/core/db.py::get_tenant_specific_metadata()` construye una copia de metadata
  de SQLAlchemy que incluye únicamente las tablas propias de un tenant, y una de ellas
  (`payment_methods`) tiene una referencia real hacia una tabla del catálogo compartido de la
  plataforma (`payment_method_catalog`, spec 032) que esa copia no incluye — así que, al preparar
  la definición de las tablas del tenant nuevo, esa referencia no puede resolverse.
- Se verificó, como parte de la auditoría previa a este spec, que esta es **hoy** la única
  referencia de este tipo (de una tabla de tenant hacia una tabla compartida) en todo el modelo
  de datos — pero FR-003 exige que la corrección cubra el caso general, no solo este caso
  puntual, para no depender de que siga siendo el único en el futuro.
- Este defecto es independiente y posterior al corregido en spec 069 (dos causas técnicas
  distintas en el mismo flujo de creación de tenant) — este spec no reabre ni modifica nada de lo
  ya corregido ahí.
- No hay datos que migrar ni limpiar: los intentos de creación de tenant que fallaron por este
  defecto no llegaron a dejar ningún dato parcial (la transacción se revierte por completo,
  verificado antes de escribir este spec).
- Este hotfix no modifica ninguna regla de negocio de creación de tenants (spec 033) ni del
  catálogo de métodos de pago (spec 032) — es una corrección de cómo el sistema prepara
  internamente la definición de las tablas al crear un tenant, acotada a esa causa.
