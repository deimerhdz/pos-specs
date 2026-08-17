# Specification Quality Checklist: Sesión de mesa, reparto entre comensales, cierre unificado/dividido y barrido de sesiones abandonadas

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
  nuevas — pero sí se citan endpoints (`POST /table-sessions/{id}/close`, `GET .../bill`, `PUT
  .../assignments`), códigos HTTP, nombres de campo, constantes internas
  (`EMPTY_SESSION_TTL_MINUTES`, `TABLE_SESSION_MAX_HOURS`) y valores literales existentes, porque
  **son** el contrato observable que la spec documenta y el criterio de verificación exigido (ver
  Assumptions y SC-001/SC-002/SC-003 en `spec.md` para los scripts de characterization citados).
- **A-15 es la regla `[PROTEGIDA]` central de esta spec** (User Story 3): cuatro huecos de
  seguridad del cobro dividido cerrados antes de dar a los cajeros la capacidad de armar bloques
  de pago manualmente, con decisión de negocio ya cerrada (P12: sin ventana de exposición). Se
  especifica tal cual, sin tocar, con `test_split_blindaje.py` como base de test obligatoria.
- **Tres anomalías se documentan explícitamente sin especificarse como contrato**, por
  instrucción de alcance del usuario y porque su clasificación en `registro-de-anomalias.md`
  sigue `PENDIENTE` o sin decisión de negocio cerrada:
  - A-17 (porción mesa, `RN-MESA-02`, User Story 7): `[DUDOSA]`, la única de las tres cuya
    pregunta de negocio **sigue sin decisión concluyente** (no se cerró ni en ronda 1-2 ni en
    ronda 3 simulada) — se marca explícitamente para la próxima ronda real.
  - A-29 (`RN-MESA-15`, User Story 11): `PENDIENTE`, sin impacto práctico confirmado (P21).
  - A-38 (`RN-MESA-13`, `RN-MESA-24`, User Story 12): `PENDIENTE`, cluster de hallazgos menores.
  Esto no es una brecha de la spec ni un `[NEEDS CLARIFICATION]` disfrazado: es una decisión
  deliberada de "documentar sin especificar" que la propia spec explica en cada User Story, en
  FR-002, FR-014, FR-023, FR-027 y en Assumptions.
- **A-11 se referencia, no se respecifica** (User Story 6): esta spec confirma que el alcance
  decidido por el negocio (ronda 3, P30) incluye el cierre unificado y dividido de mesa, pero
  delega la especificación completa del mecanismo de tope/prohibición de descuento manual a la
  spec 011, evitando duplicar una regla común a los tres caminos de cobro.
- **A-28 se documenta como riesgo de configuración no corregido por esta spec** (User Story 9,
  FR-038): el invariante `SESSION_TTL_REFRESH_SLACK_MINUTES < EMPTY_SESSION_TTL_MINUTES` no tiene
  validación de arranque; si se viola, el barrido puede cerrar mesas activas.
- **`table_sessions.compute_bill` se fija como la implementación vigente y correcta** de "cuánto
  debe una mesa" (User Story 1, nota A-01), en contraste explícito con los dos caminos
  divergentes documentados en las specs 008 (`orders/checkout.compute_bill`, código muerto) y 009
  (`tables_advanced.group_bill`, en uso real para mesas fusionadas) — sin corregir esos otros dos
  caminos, que quedan fuera del alcance de esta spec.
- Items marcados incompletos requerirían actualizar la spec antes de `/speckit-clarify` o
  `/speckit-plan`. No aplica aquí: no quedaron `[NEEDS CLARIFICATION]` porque el alcance, los
  valores concretos y el tratamiento de cada anomalía ya venían resueltos por el usuario en el
  prompt de entrada, citando `reglas-de-negocio.md`, `registro-de-anomalias.md` y
  `memoria-historica.md`.
