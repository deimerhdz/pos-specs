# Specification Quality Checklist: Resumen del Pedido en la Pantalla de Pago por Transferencia

**Purpose**: Validar la completitud y calidad de la especificación antes de pasar a planificación
**Created**: 2026-10-02
**Feature**: [spec.md](../spec.md)

## Content Quality

- [x] No implementation details (languages, frameworks, APIs) — el cuerpo normativo (historias, FR, reglas de negocio, criterios de éxito) está expresado solo en comportamiento observable. Las decisiones técnicas que el usuario ya había confirmado antes de redactar la spec están aisladas en el anexo "Decisiones de Diseño Ya Confirmadas", rotulado explícitamente como *entrada para la fase de plan, no requisitos*; se conservan ahí por el Principio XII (trazabilidad), que exige poder rastrear por qué se implementó así
- [x] Focused on user value and business needs — el problema está declarado en términos del comensal (no sabe cuánto transferir) y de su consecuencia de negocio (monto equivocado → el cajero rechaza el pago y rehace el cobro)
- [x] Written for non-technical stakeholders — el cuerpo no exige conocer el código; el anexo técnico está al final y marcado como tal
- [x] All mandatory sections completed — además de las obligatorias de la plantilla, se incluyen las que el Principio I exige: problema, reglas de negocio, impacto sobre funcionalidades existentes, impacto sobre datos existentes y decisiones de compatibilidad

## Requirement Completeness

- [x] No [NEEDS CLARIFICATION] markers remain — no se emitió ninguno: las dos decisiones que lo habrían merecido (estructura del resumen y qué hacer con impuestos/subtotal) se resolvieron con el usuario **antes** de redactar y quedaron registradas en Clarifications
- [x] Requirements are testable and unambiguous — FR-001 se reescribió en esta validación para no depender de "jerarquía visual" como juicio subjetivo: ahora declara cómo verificarlo (texto de mayor tamaño de la vista, primer dato bajo el encabezado)
- [x] Success criteria are measurable — las nueve SC traen porcentaje, conteo o medida de dispositivo
- [x] Success criteria are technology-agnostic — SC-003 y SC-009 nombran anchos de pantalla (360 px, 320 px), que son condiciones de uso del comensal, no tecnología
- [x] All acceptance scenarios are defined — 4 historias con 21 escenarios (5 + 8 + 5 + 3); la tabla de trazabilidad mapea los 8 criterios de aceptación del brief original a sus FR, SC y escenarios
- [x] Edge cases are identified — 11 casos, incluida la frontera exacta del umbral (3 vs. 4 líneas), el pedido de una sola línea, la recarga del navegador, el método sin QR y el negocio con varios métodos que exigen comprobante
- [x] Scope is clearly bounded — sección "Fuera de Alcance" con 7 exclusiones explícitas, la primera de ellas modelar impuestos
- [x] Dependencies and assumptions identified — 6 supuestos (formato de moneda existente, origen del umbral de 3 líneas, sin recarga en vivo, sin dependencias nuevas, conteo en unidades vs. umbral en líneas) más la sección de decisiones de compatibilidad

## Feature Readiness

- [x] All functional requirements have clear acceptance criteria — se corrigieron dos huecos de cobertura detectados en esta validación: FR-003/FR-006 (el conteo de productos) no tenían escenario y se añadió US1 esc. 5; FR-020 (aplica a cualquier método que exija comprobante, no solo al llamado "Transferencia") no tenía caso y se añadió como caso límite
- [x] User scenarios cover primary flows — ver el total, revisar los ítems, enviar el comprobante sin estorbo, y consistencia entre los dos pasos del checkout
- [x] Feature meets measurable outcomes defined in Success Criteria
- [x] No implementation details leak into specification — mismo criterio que el primer ítem: el anexo es deliberado y está rotulado; ningún FR, SC ni regla de negocio depende de él

## Notes

- **Iteraciones de validación**: 1. Tres hallazgos, los tres corregidos en la misma iteración (FR-001 subjetivo; FR-003/FR-006 y FR-020 sin escenario que los verificara). Ningún ítem quedó en rojo.
- **Decisión de negocio registrada en la propia spec**: no mostrar impuestos ni subtotal mientras el sistema no los calcule (RN-003, FR-013). No altera ningún comportamiento existente, así que no requiere entrada en `specs/000-reconocimiento/registro-de-anomalias.md` por el Principio II — nada que hoy funcione de una forma pasa a funcionar de otra; solo se añade información que antes no se mostraba.
- **Sin impacto en datos ni en el backend**: spec exclusivamente de presentación. No hay migración, ni estrategia de rollback de datos, ni cambio de contrato de API que planificar (Principio VIII no aplica).
- **Riesgo a vigilar en el plan**: FR-021/FR-022 convierten el resumen del paso de revisión en un artefacto compartido. Cualquier diferencia visible en ese paso respecto a hoy es una regresión, no una mejora — conviene cubrirlo con tests antes de extraerlo.
- **Pendiente deliberado para el plan**: el set de iconos del proyecto no tiene chevron. Añadirlo o rotar el indicador nativo por CSS son ambas válidas frente a los FR; es decisión de implementación.
- Lista para `/speckit-clarify` (opcional, no hay ambigüedades pendientes) o directamente `/speckit-plan`.
