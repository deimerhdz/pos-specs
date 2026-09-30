# Specification Quality Checklist: Correcciones de Adicionales del Menú QR, Cierre de Mesa, Impresión de Caja y Total Inmediato en el POS

**Purpose**: Validar la completitud y calidad de la especificación antes de pasar a planificación
**Created**: 2026-09-30
**Feature**: [spec.md](../spec.md)

## Content Quality

- [x] No implementation details (languages, frameworks, APIs) — el cuerpo de las historias y los FR describe comportamiento; los nombres de componentes y archivos solo aparecen en `Clarifications` (hallazgos del reconocimiento) y en las anomalías, igual que en las specs 087/088
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
- [x] Scope is clearly bounded (solo Menú QR para adicionales; solo el reporte de cierre de caja para impresión; recibos y hoja de QR fuera de alcance)
- [x] Dependencies and assumptions identified

## Feature Readiness

- [x] All functional requirements have clear acceptance criteria
- [x] User scenarios cover primary flows
- [x] Feature meets measurable outcomes defined in Success Criteria
- [x] No implementation details leak into specification

## Notes

- Sin marcadores `[NEEDS CLARIFICATION]`: las 17 preguntas de la sesión de aclaración del 2026-09-30 cubrieron las ambigüedades de alcance; los puntos restantes se resolvieron con valores por defecto documentados en `Assumptions`.
- **Por confirmar con el negocio** (valor por defecto ya escrito en la spec, no bloquea el plan):
  1. FR-028 / Historia 2, escenario 7: si el total cambia por una causa externa justo al cobrar, se muestra un aviso no bloqueante y se exige un segundo Cobrar (nunca se cobra un importe no visto). Registrado en A-96.
  2. FR-003: adicional = opción de un grupo **con recargo**; las opciones de grupos incluidos (sabores) siguen por unidad de producto.
  3. Assumptions: el mismo tratamiento no bloqueante se aplica al aviso del alta de pedido manual (spec 073, FR-015a).
- Anomalías registradas antes de implementar (Principio II): **A-94** (adicionales por unidades elegidas), **A-95** (notificación y pantalla de gracias al cerrar mesa), **A-96** (retiro del modal "El total cambió"). La impresión del cierre de caja (Historia 5) es un defecto, no un cambio de comportamiento autorizado, y no requiere anomalía.
- Riesgo técnico para el plan: la causa de la hoja en blanco es una hipótesis no confirmada (FR-024); y la nueva regla de adicionales exige distinguir líneas nuevas de históricas sin reescribir datos (Principios VII y VIII).
