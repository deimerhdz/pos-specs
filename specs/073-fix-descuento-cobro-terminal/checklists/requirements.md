# Specification Quality Checklist: Corrección — la Terminal de mesas cobra sin aplicar el descuento por promoción

**Purpose**: Validar la completitud y calidad de la especificación antes de pasar a planificación
**Created**: 2026-09-02
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

### Sobre las citas de código (ítems 1, 3 y 16)

La sección **"Contexto del defecto"** cita archivos, funciones y líneas de `pos-heladeria` y
`pos-backend`. Es deliberado y consistente con el precedente del repositorio para specs de
corrección (specs 019, 029, 050, 069, 072): en un defecto, el código actual **es** el
comportamiento observable que se está corrigiendo, no una decisión de implementación anticipada.

La separación se mantiene estricta:

- Las citas viven **solo** en "Contexto del defecto", una sección de diagnóstico claramente
  delimitada y precedida de la justificación en el encabezado.
- Las **Historias de usuario**, los **Requisitos funcionales** (FR-001 a FR-024, más los
  sub-requisitos con sufijo — FR-007a, FR-009a, FR-011a, FR-012a, FR-015a, FR-018a), los
  **Criterios de éxito** (SC-001 a SC-009 más SC-002a) y las **Suposiciones** están redactados en
  lenguaje de negocio, sin nombres de archivo, clase, función ni endpoint, y son legibles por el
  dueño del negocio sin abrir el código.
- Ningún FR prescribe **cómo** implementar: FR-002 exige que el monto lo calcule la misma
  autoridad que emite la venta (restricción de consistencia verificable por SC-002), no dónde ni
  con qué mecanismo hacerlo. Esa elección queda para `/speckit-plan`.

### Trazabilidad requisito → escenario de aceptación

| Requisito | Cubierto por |
|---|---|
| FR-001, FR-002 | US1 esc. 1-5, US2 esc. 1-2, US3 esc. 4 |
| FR-003 | US3 esc. 2-3, US2 esc. 3 |
| FR-004 | US1 esc. 1, US3 esc. 2-3 |
| FR-005 | US1 esc. 2-4 |
| FR-006 | US2 esc. 1-3 |
| FR-007 | Edge case "La cuenta cambia mientras el cajero teclea" y "La promoción se pausa, se borra o se agota" |
| FR-007a | SC-008 + Edge case "Sin conexión al consultar el total del cobro" |
| FR-008, FR-009 | US4 esc. 1-2 |
| FR-009a | Edge case "La promoción se pausa, se borra o se agota" |
| FR-010 | US4 esc. 3 |
| FR-011 | US4 esc. 4, SC-007 |
| FR-011a | SC-009 |
| FR-012 | Edge case "Pedido creado antes de este cambio" |
| FR-012a | US4 esc. 5 + Edge case "Rondas sucesivas en una misma mesa" |
| FR-013, FR-014 | US5 esc. 1-3 |
| FR-015 | US5 esc. 4 |
| FR-015a | US5 esc. 5, SC-005 |
| FR-016 | US6 esc. 1-2, esc. 4 |
| FR-017 | US6 esc. 2, esc. 4 |
| FR-018 | Tabla "El mismo defecto en los demás flujos" (filas marcadas Correcto) + US4 esc. 1 |
| FR-018a | US4 esc. 6 + quickstart §Verificación cruzada (mesas fusionadas) |
| FR-019, FR-020 | Alcance protegido; verificable por regresión sobre specs 029, 036 y 046 |
| FR-021 | US7 esc. 1-6 |
| FR-022 | US7 esc. 1, esc. 3-4 + Edge case "La tarjeta de 'Pagos por confirmar' muestra un total desactualizado" |
| FR-023 | US7 esc. 2, esc. 4 + SC-002a |
| FR-024 | US7 esc. 7 + Edge case "La promoción se pausa entre el pedido QR y la confirmación del cajero" |

### Decisiones registradas en la primera ronda de aclaración (2026-09-02, previa a la redacción)

Las cuatro ambigüedades materiales se resolvieron **antes** de redactar, con el dueño del
proyecto, así que la spec no lleva ningún marcador `[NEEDS CLARIFICATION]`:

1. Desglose del descuento → fila agregada (Subtotal / Descuento / Total).
2. Alcance → las cuatro superficies (cobro en terminal, cobro para llevar y domicilio, armado de
   orden manual, catálogo de la terminal).
3. Valor del domicilio faltante en el total → entra en esta misma spec.
4. Vigencia de la promoción → manda la del momento del pedido, no la del cobro.

### Decisiones registradas en la segunda ronda (`/speckit-clarify`, 2026-09-02, posterior a la redacción)

Cinco huecos que la redacción dejó abiertos alrededor del instante congelado, resueltos con el
dueño y volcados en la sección **Clarifications** de la spec:

1. Cuenta de mesa con varias rondas → un solo instante, el del pedido más antiguo pendiente de
   cobro, sin cambiar la agrupación de líneas actual (FR-012a).
2. Estado de la promoción → **no** se congela, se lee vivo; solo se congela la vigencia temporal.
   Resuelve la contradicción entre el texto original de FR-009 y el caso borde de pausa/borrado
   (FR-009, FR-009a).
3. Rendimiento del total autoritativo → ≤ 1 s en el 95%, con estado de "calculando" por encima
   (FR-007a, SC-008).
4. Vencimiento de la franja mientras se arma una orden manual → segunda confirmación con el total
   actualizado antes de crear el pedido (FR-015a, SC-005 reformulado).
5. Trazabilidad de un descuento de promoción hoy vencida → la venta emitida guarda su instante
   congelado (FR-011a, SC-009).

### Decisiones registradas en la tercera ronda (`/speckit-clarify`, 2026-09-03, tras implementar)

El dueño reprodujo el bug todavía presente en el panel **"Pagos por confirmar"** (revisión de
pago del cajero para un pedido de comensal por QR): la tarjeta muestra `$ 8.000` pero al
confirmar $10.000 en efectivo el sistema responde *"El monto recibido (10000) es menor al total
de la orden (16000.00)"*. Tres huecos resueltos con el dueño y volcados en **Clarifications**:

1. Alcance → esa revisión de pago del cajero entra como **quinta superficie**; el chequeo previo
   del backend y el total/vuelto/desglose del panel usan la misma cuenta autoritativa que emite
   la venta (FR-021–FR-024). Deroga la fila "Comensal por QR — Correcto" del análisis del defecto.
2. Tipos de orden → el panel "Pagos por confirmar" es solo de mesa (los pedidos QR se crean como
   `DINE_IN`); para llevar y domicilio se cobran en el panel de cobro, ya cubierto. Se añade
   US3 esc. 5 para dejar constancia de que el arreglo aplica a los tres tipos.
3. Total distinto al declarado (promoción pausada entre pedido y cobro) → el panel muestra el
   desglose autoritativo en vivo y, si cambió, lo marca y exige confirmación del cajero antes de
   emitir (FR-024, misma regla que FR-007/FR-007a).

No abre anomalía nueva: es el mismo defecto de FR-001/FR-002 (total sin descuento calculado
fuera de la autoridad de la venta), no detectado en esta superficie porque el total que pinta la
tarjeta venía del descuento congelado del carrito y disimulaba el fallo del chequeo del backend.

### Punto que requiere aprobación explícita antes de implementar

La decisión (4) → **FR-009** deroga una regla de negocio vigente y exige la entrada
`A-70 — [DECISIÓN DE NEGOCIO — spec 073]` en
[registro-de-anomalias.md](../../000-reconocimiento/registro-de-anomalias.md), conforme al
Principio II y al Principio XII de la Constitución (**registrada el 2026-09-02**). Es el único
cambio de la spec que **no** es restauración del comportamiento pretendido. El alcance de `A-70`
incluye los 8 call sites de
`auto_discount`/`evaluate_variant_sets` (research.md D3): `checkout.py`, `table_sessions/service.py`
y `tables_advanced.py::group_bill` (cuenta consolidada de mesas fusionadas — FR-018a). No incluye
congelar el **estado** de la promoción (FR-009a).

---

- Los ítems marcados incompletos exigirían actualizar la spec antes de `/speckit-clarify` o
  `/speckit-plan`. **Ninguno lo está**: la spec está lista para planificación.
