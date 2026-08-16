# Specification Quality Checklist: Turnos de caja, movimientos manuales y arqueo

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
  aplica a este tipo de spec: no se prescriben lenguajes ni frameworks nuevos — pero sí se citan
  endpoints, campos, fórmulas y mensajes de error existentes, porque **son** el contrato
  observable que la spec documenta y el criterio de verificación exigido.
- **Caso especial señalado explícitamente, tal como pidió el usuario al invocar
  `/speckit-specify`**: dos de las diecisiete reglas (`RN-CASH-13`/`FR-013`, User Story 6, y
  `RN-CASH-17`/`FR-017`, User Story 7) **no** documentan el comportamiento actual como contrato
  válido — documentan un requisito de negocio **nuevo y ya confirmado** en la entrevista (rondas
  1 y 2, `entrevista-negocio.md` §4), más estricto que lo que el código hace hoy. Ambas User
  Stories citan explícitamente el comportamiento actual como "el gap que este requisito cierra",
  para que no se confunda con documentación de lo existente. Los characterization tests que se
  escriban para estas dos reglas deberán capturar primero el comportamiento actual como regresión
  conocida y luego verificar el contrato nuevo — señalado como prioridad en `SC-002`.
- **`RN-CASH-09` (A-20, parte 1) se documenta tal cual, con mitigación fuera del backend**: el
  negocio confirmó (P14) que la pantalla de cierre siempre exige el conteo antes de cerrar, así
  que esta spec no agrega una validación de backend que el negocio no pidió — decisión explícita
  registrada en Assumptions, no un descuido.
- **`A-17` (porción caja, User Story 8/FR-018) se especifica como corrección técnica sin decisión
  de negocio pendiente** — a diferencia de las dos anteriores, es una inconsistencia verificable
  por contraste directo con el patrón de bloqueo de fila (`lock_items`) que el resto del sistema
  ya sigue (spec 005).
- **`A-40` (alias `cash_sales`, User Story 9/FR-015) permanece `PENDIENTE`, documentado sin fijar
  su retiro** — la pregunta de si el frontend ya migró queda abierta y registrada, sin bloquear
  esta spec.
- Items marcados incompletos requerirían actualizar la spec antes de `/speckit-clarify` o
  `/speckit-plan`. No aplica aquí: no quedaron `[NEEDS CLARIFICATION]` porque el alcance, las
  reglas concretas y el tratamiento de cada anomalía ya venían resueltos por el usuario en el
  prompt de entrada, citando `reglas-de-negocio.md` y `registro-de-anomalias.md`.
