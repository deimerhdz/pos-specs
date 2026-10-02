# Implementation Plan: Resumen del Pedido en la Pantalla de Pago por Transferencia

**Branch**: la spec vive en `specs/092-resumen-pedido-transferencia/` de `pos-specs`, rama `main`. La rama de código de `pos-heladeria` es `feat/092-payment-step-order-summary` (Principio XIV) | **Date**: 2026-10-02 | **Spec**: [spec.md](./spec.md)

**Input**: Feature specification from `/specs/092-resumen-pedido-transferencia/spec.md`

## Summary

El paso 3 del checkout del comensal (`checkout/transfer`, alcanzado por **cualquier** método de pago no-efectivo) gana dos cosas: un bloque de **total a pagar** destacado arriba de todo el contenido, y una sección **"Resumen del pedido"** colapsable con las líneas del pedido. El resumen no se escribe nuevo: se **extrae** el que ya vive embebido en el paso 1 (`review-step.component.ts`) a un componente presentacional propio que ambos pasos consumen, con el colapso apagado en el paso 1 (FR-021/FR-022).

El enfoque técnico es deliberadamente mínimo: un componente presentacional nuevo (`checkout-order-summary`), un icono nuevo (`chevron-down`) en el set propio del proyecto, y dos campos nuevos **aditivos** en `DiningCartService` (`grossTotal` y `savings`) para poder pintar la fila "Ahorro" que hoy el servicio descarta. Cero cambios de backend, cero cambios de contrato de API, cero dependencias nuevas, cero migraciones. El colapsable es `<details>`/`<summary>` nativo, lo que hace que FR-010 (expandir no dispara nada) y FR-011 (teclado + lector de pantalla) se cumplan por construcción en vez de por código.

## Technical Context

**Language/Version**: TypeScript 5.9 · Angular 21.1 (standalone components, signals, control flow `@if`/`@for`)

**Primary Dependencies**: `@angular/core` 21.1, `@angular/router` 21.1, Tailwind CSS 4.1 (vía `@import "tailwindcss"` en `src/styles.css`). **Ninguna dependencia nueva** (Principio IX): el colapsable es HTML nativo y el icono se añade al set propio (`src/app/shared/icon/icon.component.ts`). `@angular/cdk` 21.2 ya está en el proyecto y se evaluó como alternativa — ver [research.md](./research.md) D1.

**Storage**: N/A. Spec exclusivamente de presentación. No hay entidad, campo, relación ni migración (Principio VIII no aplica). El dato se lee del estado ya cargado de `DiningCartService`, que se alimenta de `GET /cart` sin cambios de contrato.

**Testing**: Vitest 4.0 ejecutado por el builder `@angular/build:unit-test` (`npm test` → `ng test`), con `TestBed` + jsdom 27. Convención del repo: un `*.component.spec.ts` hermano por componente.

**Target Platform**: navegador móvil del comensal (Chrome/Safari en teléfono), entrada por QR de mesa. Anchos objetivo: 320 px mínimo (SC-009) y 360 × 640 px como referencia de medición (SC-003).

**Project Type**: aplicación web — frontend Angular (`pos-heladeria`) consumidor de la API. **Solo el frontend cambia**; `pos-backend` no se toca.

**Performance Goals**: N/A en el sentido clásico. La restricción de rendimiento real es de *layout*, no de CPU: la acción "Enviar pedido" debe alcanzarse con a lo sumo un desplazamiento de pantalla en 360 × 640 px con 6 líneas (SC-003/FR-017). No se añade ninguna petición de red (FR-010).

**Constraints**:
- El total mostrado MUST salir del mismo cálculo que ya alimenta el paso 1 — ninguna cifra se recalcula en el front (FR-014/RN-001).
- Expandir/contraer MUST NOT disparar red ni mutar el carrito (FR-010).
- El flujo de comprobante (adjuntar / reemplazar / quitar / enviar / rehidratar) MUST seguir idéntico (FR-018).
- Ninguna fila de impuestos ni de subtotal, ni en cero (FR-013/RN-003).
- Sin desplazamiento horizontal desde 320 px (FR-019/SC-009).

**Scale/Scope**: 2 pantallas tocadas (`review-step`, `transfer-details-step`), 1 componente nuevo, 1 servicio extendido de forma aditiva, 1 icono nuevo. ~5 archivos de producción y ~3 de test en `pos-heladeria`. Cero archivos en `pos-backend`.

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| Principio | Estado | Evidencia |
|-----------|--------|-----------|
| **I** — Las nuevas funcionalidades nacen de un spec | ✅ PASA | `spec.md` con 23 FR, 4 RN, 9 SC, impacto sobre funcionalidades y datos, y decisiones de compatibilidad. Validado con `checklists/requirements.md` (cero ítems en rojo). |
| **II** — El comportamiento existente sigue protegido | ⚠️ PASA CON ACCIÓN | Dos comportamientos existentes cambian de forma visible: (a) el `<h1>` con el nombre del método del paso 3 baja por debajo del bloque de total (FR-001 lo exige explícitamente); (b) el paso 1 gana la fila "Ahorro" cuando hay promoción vigente. (a) está documentado en la spec sin ambigüedad. (b) es el único punto donde la spec se contradice consigo misma (FR-012+FR-021+US4 esc.1 la exigen; la sección "Impacto sobre Funcionalidades Existentes" dice "cambia de forma estructural, no visual"). **Acción**: resuelto en [research.md](./research.md) D6 a favor de mostrarla en ambos pasos, y registrado como pendiente de confirmación del negocio antes de implementar US4. Si el negocio decide lo contrario, la salida es un input del componente, no un rediseño. |
| **III** — Los characterization tests protegen el comportamiento heredado | ✅ PASA | Ningún test `"CONGELA comportamiento actual:"` se toca. Los de `pos-backend` (`test_cart_service.py`, que fijan `discounted_total < total` y `None` sin promoción) quedan intactos y son precisamente el contrato del que sale el "Ahorro" (research.md D5). |
| **IV** — Los nuevos specs pueden introducir nuevo comportamiento | ✅ PASA | El bloque de total y el resumen son comportamiento nuevo autorizado por FR-001 a FR-011. |
| **V** — Nuevas funcionalidades antes que refactorizaciones oportunistas | ✅ PASA | La única extracción de código (el resumen del paso 1 → componente propio) **no es oportunista**: la exige FR-021 y la justifica RN-004 (una sola fuente para que los dos pasos no divergan). `cart.component.ts` repite el mismo markup de líneas y **no se toca** — queda fuera de alcance por decisión explícita (research.md D7), no por olvido. |
| **VI** — Evolución incremental | ✅ PASA | Las 4 historias son entregables independientes y en ese orden: US1 (total) → US2 (resumen colapsable) → US3 (no estorbar el comprobante) → US4 (extracción compartida). US1 entrega valor sola. La extracción (US4), que es la única parte con riesgo de regresión, va al final y con tests de base previos. |
| **VII** — Compatibilidad con datos históricos | ✅ PASA | No se escribe nada. No se toca ninguna factura, ni su importe ni su representación. El resumen es de lectura sobre el carrito borrador, que todavía no es pedido ni factura. |
| **VIII** — Evolución del modelo de datos | ✅ N/A | Cero entidades, campos, relaciones, migraciones y rollback de datos. Ver [data-model.md](./data-model.md), que documenta solo *modelo de vista*. |
| **IX** — Dependencias nuevas con justificación | ✅ PASA | Cero dependencias nuevas, conforme al supuesto de la spec. `@angular/cdk` (ya presente) se consideró y se descartó frente a `<details>` nativo — research.md D1. |
| **X** — Verificación obligatoria | ✅ PASA | Tests unitarios por componente (Vitest + TestBed) para FR-001 a FR-015 y FR-021/FR-022, más recorridos manuales en dispositivo para lo que un test de jsdom no puede medir: desplazamiento real (SC-003), ancho de 320 px (SC-009) y anuncio de lector de pantalla (SC-006). Detalle en [quickstart.md](./quickstart.md). |
| **XI** — Decisiones de negocio frente a decisiones técnicas | ✅ PASA | Separadas explícitamente: el umbral de 3 líneas y la fila "Ahorro" en el paso 1 son decisiones de **negocio** (research.md D6, D8); el mecanismo del colapsable, el chevron y dónde vive el componente son decisiones **técnicas** (D1 a D4). |
| **XII** — Trazabilidad | ✅ PASA | Cadena completa: brief del usuario → `spec.md` (con tabla de trazabilidad de los 8 criterios del brief) → este plan + `research.md` → `tasks.md` → tests → `quickstart.md`. Cada decisión de research cita el FR que la obliga. |
| **XIII** — Todo en español de Colombia | ✅ PASA | Todos los artefactos de esta spec, los comentarios de código nuevo y los nombres de test se escriben en español de Colombia. Mensajes de commit y nombre de rama en inglés (excepción de los Principios XIV/XV). |
| **XIV** — Estrategia y convención de ramas | ✅ PASA | `feat/092-payment-step-order-summary` en `pos-heladeria`, creada **antes** de tocar código. `pos-backend` no se toca, así que no lleva rama. |
| **XV** — Política de commits | ✅ PASA | Ningún commit sin pedido explícito. Cuando se pida: unidades pequeñas (icono · servicio · componente nuevo + test · paso 3 · extracción del paso 1), en inglés, Conventional Commits, sin marcas de autoría de IA. |

**Veredicto de la compuerta**: **PASA**. El único ⚠️ (Principio II, fila "Ahorro" en el paso 1) no bloquea el diseño: está resuelto en el diseño y aislado detrás de un input del componente, de modo que la decisión del negocio se materializa cambiando un `false` por un `true` en una plantilla, sin rediseñar nada. Queda como tarea de confirmación en `tasks.md`, no como deuda silenciosa.

**Re-evaluación post-Phase 1**: sin cambios. El diseño no introdujo ninguna violación nueva: no apareció ninguna dependencia, el modelo de datos sigue vacío, el contrato del componente no obliga a tocar el backend, y la fila "Ahorro" quedó aislada tras el input `showSavings`. Ver "Complexity Tracking" (vacío).

## Project Structure

### Documentation (this feature)

```text
specs/092-resumen-pedido-transferencia/
├── spec.md                    # Especificación funcional (ya existente)
├── plan.md                    # Este archivo (salida de /speckit-plan)
├── research.md                # Fase 0 — D1 a D11
├── data-model.md              # Fase 1 — modelo de vista (sin cambios de persistencia)
├── quickstart.md              # Fase 1 — guía de verificación ejecutable
├── contracts/
│   └── checkout-order-summary.md   # Contrato del componente compartido
├── checklists/
│   └── requirements.md        # Validación de calidad de la spec (ya existente)
└── tasks.md                   # Fase 2 — la crea /speckit-tasks, NO este comando
```

### Source Code (repository root)

Un solo repositorio cambia: `pos-heladeria` (frontend Angular). `pos-backend` no se toca.

```text
pos-heladeria/src/app/
├── modules/tables/
│   ├── pages/checkout/
│   │   ├── checkout-order-summary.component.ts        # NUEVO — componente presentacional compartido
│   │   ├── checkout-order-summary.component.spec.ts   # NUEVO — FR-005..FR-015, FR-021/FR-022
│   │   ├── checkout-product-count.ts                  # NUEVO — composición única de la cadena del conteo
│   │   ├── checkout-product-count.spec.ts             # NUEVO — singular/plural de FR-003
│   │   ├── review-step.component.ts                   # CAMBIA — consume el componente, sin colapso
│   │   ├── review-step.component.spec.ts              # NUEVO — base de no-regresión del paso 1
│   │   ├── transfer-details-step.component.ts         # CAMBIA — bloque de total + resumen colapsable
│   │   └── transfer-details-step.component.spec.ts    # CAMBIA — amplía el mock de DiningCartService
│   └── services/
│       ├── dining-cart.service.ts                     # CAMBIA — añade `grossTotal` y `savings` (aditivo)
│       └── dining-cart.service.spec.ts                # CAMBIA — cubre `savings` con y sin promoción
└── shared/icon/
    └── icon.component.ts                              # CAMBIA — añade el caso `chevron-down`
```

**Desvío registrado durante la implementación (T010)**: se añadió `checkout-product-count.ts` — un sexto archivo de producción que este plan no previó — con la función pura `formatProductCount`. Razón: FR-006 exige que el bloque de total y el `<summary>` del resumen usen **una sola** composición de la cadena del conteo, y US1 debe poder entregarse **antes** de que exista el componente compartido (Principio VI). Si la composición viviera solo en el componente, US1 dependería de US2 y dejaría de ser un incremento independiente. Cumple la intención de la invariante 2 de [contracts/checkout-order-summary.md](./contracts/checkout-order-summary.md), que quedó corregida para nombrarla.

**Structure Decision**: el componente nuevo vive en `modules/tables/pages/checkout/`, junto a los pasos que lo consumen, siguiendo el precedente de `checkout-step-indicator.component.ts` — el otro componente que ya se comparte entre pasos del checkout y que por eso no se subió a `modules/tables/components/`. Esa carpeta de `components/` se reserva, por convención viva del repo, a lo que consumen pantallas de **módulos o roles distintos** (terminal del cajero, panel de mesas, menú público); el resumen del checkout solo lo consumen dos rutas hermanas. El nombre lleva el prefijo `checkout-` porque `modules/tables/components/order-summary-card.component.ts` ya existe y es otra cosa (tarjeta de pedido del cajero).

## Complexity Tracking

> Fill ONLY if Constitution Check has violations that must be justified

Sin violaciones que justificar. El único ⚠️ del Constitution Check es una **contradicción interna de la spec** resuelta en research.md D6 y pendiente de confirmación del negocio, no una desviación de un principio ni una complejidad añadida al diseño.
