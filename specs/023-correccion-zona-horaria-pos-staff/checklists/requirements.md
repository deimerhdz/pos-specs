# Specification Quality Checklist: Corrección de zona horaria en el POS de staff (previsualización de promociones) (A-09)

**Purpose**: Validate specification completeness and quality before proceeding to planning
**Created**: 2026-08-18
**Feature**: [spec.md](../spec.md)

## Content Quality

- [x] No implementation details (languages, frameworks, APIs) — *excepción documentada en
  Assumptions: esta es una spec delta de corrección sobre comportamiento ya observado, igual que
  las specs 007/012/020/021/022; cita nombres de función, archivo y línea porque son el contrato
  observable que se corrige, no una fuga de detalles de implementación. El mecanismo exacto de la
  hora sincronizada con el servidor se deja explícitamente para `plan.md` (FR-001/FR-002,
  Assumptions), no se prescribe aquí.*
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

- No hubo marcadores `[NEEDS CLARIFICATION]`, pero esta spec sí toma una decisión que las tres
  correcciones anteriores (A-04, A-44, A-08) no tuvieron que tomar: **reabre una decisión de
  negocio previamente cerrada** (A-09 estaba "mitigado operativamente, documentar sin especificar",
  no "corregir en modernización"). Esa reapertura queda documentada explícitamente en el preámbulo
  de la spec ("Autorización de negocio") y en `FR-007`, en vez de tratarse como un supuesto
  silencioso — es una decisión del propietario del repositorio, no una inferencia de la IA.
- A diferencia de A-08 (que sí tenía un test de characterization previo que actualizar, spec 015),
  **A-09 no tiene ningún test previo** que congele el comportamiento defectuoso — se verificó
  directamente en `pos-terminal.store.spec.ts` y `promotion-pricing.util.spec.ts`. Por eso `FR-008`
  exige *crear* cuatro tests nuevos (uno por punto de invocación), no actualizar uno existente.
- Se decidió explícitamente **no** tocar `promotion-pricing.util.ts` (`isPromoActiveNow`,
  `inTimeWindow`, `bestProductDiscount`) ni el motor de promociones del backend — el defecto está
  aislado a los cuatro puntos donde `pos-terminal.store.ts` construye `now` con el reloj del
  dispositivo. Esta decisión de alcance está documentada en Out of Scope y Assumptions, no es un
  `[NEEDS CLARIFICATION]`.
- El panel de administración de promociones (`promotions-page.component.ts`) tiene la misma clase
  de defecto pero se dejó fuera de alcance explícitamente (superficie back-office distinta, no POS
  de venta) — documentado en Out of Scope y Edge Cases, disponible como candidato a una delta
  separada si el negocio lo prioriza más adelante.
- La excepción de "no implementation details" está justificada y documentada explícitamente en la
  sección Assumptions del spec, siguiendo el mismo patrón ya usado en las specs 007, 012, 015, 020,
  021 y 022 de este repositorio (correcciones sobre comportamiento ya observado, no características
  nuevas).
