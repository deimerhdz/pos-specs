# Specification Quality Checklist: Administración de promociones y combos — creación, edición, validación de forma y máquina de estados

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

- Esta es una spec de **ingeniería inversa/characterization**, no de una feature nueva —
  complementaria a la spec 012 (motor de cálculo), documentando ahora el dominio de
  administración: creación, edición, validación de forma y máquina de estados. Los ítems
  marcados de otro modo como "sin detalles de implementación" se cumplen en el sentido en que
  aplica a este tipo de spec: se citan nombres de función (`change_status`, `update_shape`,
  `duplicate`, `ensure_unique`), constantes internas (`PROMOTION_TRANSITIONS`, `AUTO_TYPES`),
  estados y valores literales existentes porque **son** el contrato observable que la spec
  documenta y el criterio de verificación exigido (ver Assumptions y SC-001/SC-002 en `spec.md`).
- **A-30 (User Story 7) es el hallazgo central de esta spec, no solo una entre varias**: a
  diferencia del resto de anomalías documentadas aquí (casos límite de configuración sin
  impacto demostrado), A-30 describe un **500 real no controlado** disparable por cualquier
  admin desde el panel — el propio prompt de entrada lo señala como "primer candidato a test
  explícito de esta spec" (ver SC-003). Sus dos vectores tienen clasificación distinta: vector 1
  (`name=null`, RN-PROMO-75) es `ACCIDENTAL`, con evidencia directa de que un bug equivalente ya
  fue corregido sin cubrir este caso; vector 2 (`targets` duplicados, RN-PROMO-76) es
  `PENDIENTE`, porque depende de un hecho no verificado en este reconocimiento (manejo genérico
  de `IntegrityError` fuera del módulo).
- **Dos anomalías se documentan explícitamente sin especificarse como contrato**, por
  instrucción de alcance del usuario y porque su clasificación en `registro-de-anomalias.md`
  sigue `PENDIENTE`:
  - A-37, porción administración (User Story 8): `PENDIENTE` — no-op silencioso al reenviar el
    mismo estado (incluido `finished→finished`), y creación directa en `active`/`paused` sin
    pasar por `draft`. Complementa las cinco porciones de cálculo ya documentadas en la spec 012,
    User Story 10.
  - A-39, ya introducida en la spec 012 desde el lado del motor de cálculo, se documenta aquí
    (User Story 9) desde el lado del job en sí: `ACCIDENTAL`, sin riesgo económico confirmado
    porque el propio comentario del job se autodescribe "puramente informativo" — con el matiz
    de precisión explicitado en Assumptions sobre el filtro SQL parcial de vigencia.
  Esto no es una brecha de la spec ni un `[NEEDS CLARIFICATION]` disfrazado: es una decisión
  deliberada de "documentar sin especificar" que la propia spec explica en cada User Story, en
  FR-015, FR-027, FR-028, y en Assumptions.
- **Ningún script de test cubre hoy la máquina de estados ni la validación de forma del
  `PATCH`/`PATCH /shape`** (SC-002): se verificó por inspección que `test_promotions_rules.py`
  (único script que corre en CI) limita su cobertura a `_valid_now`/`_in_time_window`
  (vigencia, dominio de la spec 012) — ningún `check()` ejercita `change_status`, `update_shape`
  ni `duplicate`. Gap de caracterización explícito, no ausencia de comportamiento observado.
- Items marcados incompletos requerirían actualizar la spec antes de `/speckit-clarify` o
  `/speckit-plan`. No aplica aquí: no quedaron `[NEEDS CLARIFICATION]` porque el alcance, los
  valores concretos y el tratamiento de cada anomalía ya venían resueltos por el usuario en el
  prompt de entrada, citando directamente `reglas-de-negocio.md` (RN-PROMO-46 a RN-PROMO-78,
  RN-SCHED-10/11) y `registro-de-anomalias.md` (A-30, A-37, A-39).
