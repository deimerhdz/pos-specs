# Specification Quality Checklist: Corrección de zona horaria en vigencia de promociones del menú y carrito QR (A-08)

**Purpose**: Validate specification completeness and quality before proceeding to planning
**Created**: 2026-08-18
**Feature**: [spec.md](../spec.md)

## Content Quality

- [x] No implementation details (languages, frameworks, APIs) — *excepción documentada en
  Assumptions: esta es una spec delta de corrección sobre comportamiento ya observado, igual que
  las specs 007/015/020/021; cita nombres de función, archivo y línea porque son el contrato
  observable que se corrige, no una fuga de detalles de implementación.*
- [x] Focused on user value and business needs
- [x] Written for non-technical stakeholders
- [x] All mandatory sections completed

## Requirement Completeness

- [x] No [NEEDS CLARIFICATION] markers remain
- [x] Requirements are testable and unambiguous
- [x] Success criteria are measurable
- [x] Success criteria are technology-agnostic (no implementation details) — *misma excepción
  documentada que en Content Quality, consistente con el resto de specs de este repositorio.*
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

- No hubo marcadores `[NEEDS CLARIFICATION]`: la corrección (pasar un `datetime` aware en los dos
  puntos de invocación de `active_discount_promotions`), su alcance y su carácter no retroactivo ya
  estaban decididos y citados con evidencia directa en `registro-de-anomalias.md` (A-08, por
  contraste directo con A-07) y en la propia spec 007 (`FR-030`), que dejó el comportamiento
  documentado pero explícitamente sin corregir.
- Se decidió explícitamente **no** tocar la función compartida `_now()` de `cart/service.py` (usada
  también para `expires_at` de la sesión del comensal) — solo el punto de invocación dentro de
  `serialize_cart`, para no introducir una regresión ajena a A-08 (Principio III). Esta decisión de
  alcance está documentada en Assumptions, no es un `[NEEDS CLARIFICATION]`.
- La excepción de "no implementation details" está justificada y documentada explícitamente en la
  sección Assumptions del spec, siguiendo el mismo patrón ya usado en las specs 007, 015, 020 y 021
  de este repositorio (correcciones sobre comportamiento ya observado, no características nuevas).
