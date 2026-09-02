# Specification Quality Checklist: Promociones legibles y precios reales en el menú QR

**Purpose**: Validar la completitud y la calidad de la especificación antes de pasar a planificación
**Created**: 2026-09-01
**Feature**: [spec.md](../spec.md)

## Content Quality

- [x] No implementation details (languages, frameworks, APIs)
- [x] Focused on user value and business needs
- [x] Written for non-technical stakeholders
- [x] All mandatory sections completed

## Requirement Completeness

- [x] No [NEEDS CLARIFICATION] markers remain
- [x] Requirements are testable and unambiguous
- [x] Success criteria are measurable
- [x] Success criteria are technology-agnostic (no implementation details)
- [x] All acceptance scenarios are defined
- [x] Edge cases are identified
- [x] Scope is clearly bounded
- [x] Dependencies and assumptions identified

## Feature Readiness

- [x] All functional requirements have clear acceptance criteria
- [x] User scenarios cover primary flows
- [x] Feature meets measurable outcomes defined in Success Criteria
- [x] No implementation details leak into specification

## Notes

- **Desviación deliberada en "No implementation details"**: las secciones *Estado actual* y
  *Cambios de comportamiento respecto de producción* nombran ficheros, funciones y tests
  concretos de `pos-backend` y `pos-heladeria`. No es fuga de diseño: los Principios II y III
  de la Constitución exigen que una spec que cambia comportamiento en producción liste
  explícitamente qué cambia y qué tests se ven afectados. Es la misma convención que sigue la
  [spec 063](../../063-promociones-por-variante/spec.md). Los requisitos (FR-001 a FR-020) y
  los criterios de éxito (SC-001 a SC-007) sí están redactados sin referencias de
  implementación.
- **Numeración**: la spec se creó con el número **066** por petición explícita del usuario.
  Los directorios `064-*` y `065-*` no existen en `specs/`; la secuencia queda con ese hueco a
  propósito.
- **Pendiente antes de implementar** (Principio II): registrar las decisiones de negocio
  **A-66**, **A-67** y **A-68** en `specs/000-reconocimiento/registro-de-anomalias.md` (la
  spec las propone redactadas en la sección *Cambios de comportamiento respecto de
  producción*). Sin ese registro, el cambio de comportamiento no está autorizado.
- **Verificación de tests congelados**: se comprobó que ninguno de los cuatro tests afectados
  lleva el prefijo `"CONGELA comportamiento actual:"`, así que el Principio III no exige
  autorización adicional para actualizarlos — pero sí que esta spec los nombre, como hace.
