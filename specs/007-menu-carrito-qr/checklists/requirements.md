# Specification Quality Checklist: Menú público y carrito del comensal (flujo QR)

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

- **Excepción deliberada a "No implementation details"**: esta spec es de ingeniería inversa
  (characterization spec), no de una feature nueva. Cita nombres de campo, valores de
  configuración por defecto, funciones, mensajes de error y códigos HTTP a propósito, porque son
  el contrato observable que documenta — ver la sección "Assumptions" del spec, primer punto.
  Esta excepción sigue el mismo patrón ya adoptado en las specs 002-006 de este mismo repositorio.
- Todas las reglas de negocio (`RN-MENU-01` a `RN-MENU-09`, `RN-CART-01` a `RN-CART-27`) y las
  seis anomalías del alcance (A-08, A-21, A-24, A-28, A-36, A-47) quedaron citadas con su
  evidencia de código y, donde aplica, su tratamiento acordado o clasificación confirmada por
  entrevista de negocio (P15, P27-bis).
- Gap de caracterización documentado en SC-006: no existe hoy characterization test dedicado para
  el menú público ni para el CRUD de carrito (`add_item`/`update_item`/`submit_cart`/
  `cancel_my_order`); solo la capa de sesión/token (`test_qr_token.py`,
  `test_table_sessions.py`) tiene cobertura hoy.
