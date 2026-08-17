# Specification Quality Checklist: Cocina, consolidación de carritos y mesas físicas

**Purpose**: Validate specification completeness and quality before proceeding to planning
**Created**: 2026-08-16
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

- Esta es una spec de **ingeniería inversa/characterization**, no de una feature nueva. Los
  ítems marcados de otro modo como "sin detalles de implementación" se cumplen en el sentido en
  que aplica a este tipo de spec: no se prescriben lenguajes, frameworks ni decisiones de diseño
  nuevas — pero sí se citan endpoints (`POST /orders/tables/{table_id}/items`, `.../consolidate`,
  `PATCH /orders/items/{item_id}/kitchen`, `.../void`, `POST /orders/{id}/move`, `.../merge`),
  funciones (`transition_kitchen`, `void_item`, `consolidate_table`, `add_item_to_table`,
  `move_order`, `merge_orders`), códigos HTTP, nombres de campo, y valores literales existentes,
  porque **son** el contrato observable que la spec documenta y el criterio de verificación
  exigido (ver Assumptions y SC-001/SC-002 en `spec.md`).
- **A-04 (User Story 1, `RN-CAT-33`/FR-021) es el hallazgo de mayor prioridad de toda esta spec y
  de todo el reconocimiento**: es el único con evidencia directa de `git log`/`git show` de cómo
  y cuándo se rompió (regresión de fusión entre `03469ca` y `ee94f30`), reforzado con testimonio
  de negocio real (P4, merma confirmada en sabores/toppings hace ~15 días). Se especifica con la
  corrección exacta de una línea, marcando explícitamente que **no es retroactiva**.
- **Tres anomalías cerradas en la segunda ronda de entrevista de negocio se documentan con su
  resolución, no como preguntas abiertas**: A-16/`RN-ORD-37` (P16-bis: mitigado por el ritmo de
  trabajo real), A-26/`RN-ORD-58` (P20-bis: no se usa la función de mover pedidos, riesgo
  latente), y A-48 (P28: fusión de KDS y terminal de mesas confirmada como decisión operativa
  deliberada). Las porciones ya `ACCIDENTAL` de A-16 y A-26 (inconsistencias de código
  verificables sin necesitar testigo de negocio) no cambian con esos cierres y se mantienen como
  correcciones recomendadas sin urgencia operativa (User Story 5, User Story 8, FR-004, FR-005,
  FR-028, FR-031).
- Items marcados incompletos requerirían actualizar la spec antes de `/speckit-clarify` o
  `/speckit-plan`. No aplica aquí: no quedaron `[NEEDS CLARIFICATION]` porque el alcance, los
  valores concretos y el tratamiento de cada anomalía ya venían resueltos por el usuario en el
  prompt de entrada, citando `reglas-de-negocio.md`, `registro-de-anomalias.md`,
  `entrevista-negocio.md` (P4, P16-bis, P20-bis, P28) y `propuesta-particion-specs.md`.
