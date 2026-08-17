# Specification Quality Checklist: Motor de evaluación de promociones y combos — vigencia, mejor promoción por línea, y expansión de combo

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

- Esta es una spec de **ingeniería inversa/characterization**, no de una feature nueva. Los ítems
  marcados de otro modo como "sin detalles de implementación" se cumplen en el sentido en que
  aplica a este tipo de spec: no se prescriben lenguajes, frameworks ni decisiones de diseño
  nuevas — pero sí se citan nombres de función (`local_now`, `_valid_now`, `_line_discount`,
  `combo_discount_for_lines`, `expand_combo`), tipos de promoción, constantes internas
  (`TENANT_TIMEZONE`, `AUTO_TYPES`) y valores literales existentes, porque **son** el contrato
  observable que la spec documenta y el criterio de verificación exigido (ver Assumptions y
  SC-001/SC-002 en `spec.md` para el script de characterization citado).
- **A-07 es la regla `[PROTEGIDA]` central de esta spec** (User Story 1): tres cambios
  estructurales de una reescritura completa del motor (hora local del tenant, desglose por línea,
  prioridad explícita en el desempate), respaldada por dos testigos (CÓDIGO + `memoria-historica.md`
  #15) y por el único script de test que corre en CI (`test_promotions_rules.py`). Se especifica
  tal cual, sin tocar. La pregunta de gobernanza sobre el autor no identificable del commit
  `2e94a3ad` y la pregunta de negocio sobre un posible incidente real anterior al 2026-08-07 quedan
  registradas como abiertas, sin bloquear la especificación.
- **Seis anomalías se documentan explícitamente sin especificarse como contrato**, por instrucción
  de alcance del usuario y porque su clasificación en `registro-de-anomalias.md` es `ACCIDENTAL`
  sin urgencia de corrección o sigue `PENDIENTE`:
  - A-08 (User Story 7): `ACCIDENTAL`. Se referencia, no se respecifica — la corrección
    (`cart/service.py`, `menu/router.py`) pertenece a la spec 007.
  - A-09 (User Story 8): `PENDIENTE`, mitigada operativamente por P6 (relojes de terminal
    verificados y fijados a `America/Bogota`) — riesgo de código sin corregir, sin incidente
    activo.
  - A-10 (User Story 9): `ACCIDENTAL`, sin efecto visible hoy (ninguna pantalla expone el nombre
    de la promoción ganadora en un empate).
  - A-37, porción evaluación (User Story 10): `PENDIENTE`, cinco hallazgos de configuración y
    casos límite sin impacto económico demostrado.
  - A-46 (User Story 11): `ACCIDENTAL`, cerrada sin urgencia por P26 (sin planes de expansión a
    otra zona horaria).
  - A-36, porción promociones (User Story 12): `PENDIENTE`, tres casos límite de precisión sin
    confirmación de negocio ni cobertura completa de test.
  Esto no es una brecha de la spec ni un `[NEEDS CLARIFICATION]` disfrazado: es una decisión
  deliberada de "documentar sin especificar" que la propia spec explica en cada User Story, en
  FR-039 a FR-042, FR-007, FR-017, FR-022, FR-030, FR-031, FR-037, y en Assumptions.
- **`test_promotions_rules.py` es el candidato más maduro de todo el reconocimiento a convertirse
  en golden master formal** (SC-002): único de los 12 scripts de test que corre en CI
  (`.github/workflows/deploy.yml:14-22`), cubriendo explícitamente la vigencia en hora local (no
  UTC) y la ventana con cruce de medianoche — la base directa de las User Stories 1 y 2.
- Items marcados incompletos requerirían actualizar la spec antes de `/speckit-clarify` o
  `/speckit-plan`. No aplica aquí: no quedaron `[NEEDS CLARIFICATION]` porque el alcance, los
  valores concretos y el tratamiento de cada anomalía ya venían resueltos por el usuario en el
  prompt de entrada, citando `reglas-de-negocio.md`, `registro-de-anomalias.md`,
  `memoria-historica.md` y `contradiccion-01-motor-promociones-frontend-backend.md`.
