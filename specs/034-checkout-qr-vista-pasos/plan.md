# Implementation Plan: Vista de Pasos para Revisión y Pago del Menú QR

**Branch**: `034-checkout-qr-vista-pasos` | **Date**: 2026-08-24 | **Spec**: [spec.md](./spec.md)

**Input**: Especificación de la funcionalidad en `specs/034-checkout-qr-vista-pasos/spec.md`

## Summary

Reemplazar el modal de revisión de pedido y pago del menú QR por una vista propia organizada en
pasos, que sobreviva a una recarga de página (en el mismo dispositivo/navegador), muestre el código
QR de pago del tenant como imagen (no como texto) y reemplace los iconos de emoji por el sistema de
iconos vectoriales ya existente en la aplicación.

Enfoque técnico: el progreso recuperable (paso, método elegido, referencia del comprobante ya
subido) se guarda en `localStorage` del navegador del comensal — el mismo mecanismo que ya usa el
token de sesión (spec 007, A-21) — sin requerir infraestructura nueva en el backend ni migración de
datos. El único cambio de backend es agregar al endpoint que el comensal ya consulta
(`GET /cart/payment-methods`) la metadata de formato de campo (`text`/`numeric`/`image`) que el
catálogo de métodos de pago ya guarda (spec 032) pero que hoy no llega a la respuesta del comensal.
La vista de pasos se implementa como una ruta propia (Angular, carga perezosa) en lugar de un modal,
y el reemplazo de iconos extiende el componente compartido `app-icon` ya usado en el resto de la
aplicación.

## Technical Context

**Language/Version**: Backend: Python 3.14 (FastAPI 0.136.3). Frontend: TypeScript 5.9 / Angular
21.1 (componentes standalone, signals) — ambos ya en producción, sin cambio de versión.

**Primary Dependencies**: Backend: FastAPI + Pydantic (esquemas ya existentes de
`app/api/v1/cart/schemas.py` y `app/api/v1/super_admin/schemas.py`) — sin dependencias nuevas.
Frontend: Angular Router (ya usado por `app.routes.ts`) y el componente compartido `app-icon`
(`shared/icon/icon.component.ts`) — sin librerías nuevas; se reutiliza el mismo patrón de
`localStorage` que ya usa `diner-token.store.ts` para el token de sesión.

**Storage**: Backend: PostgreSQL 16, schema-per-tenant (sin migraciones — el campo `format` de
`PaymentMethodCatalog.fields` ya existe desde spec 032; solo se expone en una respuesta que ya
existe). Frontend: `localStorage` del navegador del comensal, acotado por dispositivo/navegador
(Clarification Session 2026-08-24) y por la vigencia de la sesión ya definida en spec 007.

**Testing**: Backend: scripts autoejecutables bajo `app/scripts/test_*.py` (no hay pytest en el
proyecto; convención ya usada por `test_qr_token.py`, ejecutados con
`python -m app.scripts.test_X`). Frontend: Vitest vía el builder `@angular/build:unit-test`
(`ng test`).

**Target Platform**: Web — navegador móvil del comensal en el flujo de mesa por QR, consumiendo la
API ya en producción de `pos-backend`.

**Project Type**: Aplicación web ya existente (frontend + backend, dos repos en producción); esta
funcionalidad no agrega ningún repositorio ni servicio nuevo.

**Performance Goals**: Sin metas nuevas — la vista de pasos debe sentirse tan inmediata como el
modal actual (misma cantidad de llamadas de red que hoy: catálogo de métodos, presign+subida de
comprobante, envío del carrito); leer/escribir `localStorage` no introduce latencia perceptible.

**Constraints**: El progreso recuperable vive únicamente en el navegador del comensal (Clarification
Session 2026-08-24) — no se introduce sesión ni estado nuevo en el backend para este propósito; su
vigencia máxima está acotada por la sesión de comensal ya vigente (spec 007, FR-009 de esta spec).

**Scale/Scope**: Cambio acotado a la vista de revisión y pago del flujo QR (`pos-heladeria`) y a un
campo adicional en la respuesta de métodos de pago del comensal (`pos-backend`); no toca cocina,
caja, facturación, ni la administración del catálogo de métodos de pago.

## Constitution Check

*GATE: Debe pasar antes de la Fase 0. Se re-evalúa después del Diseño de Fase 1.*

| Principio | Evaluación |
|---|---|
| I. Nace de un spec | ✅ Cumple — spec 034 aprobada (con clarificaciones resueltas) antes de este plan. |
| II. Comportamiento existente protegido | ✅ Cumple — ninguna regla de negocio de pago de specs 024/025 cambia; "resumir tras recargar" es capacidad **nueva** (Principio IV), no la reversión de un comportamiento ya documentado. |
| III. Characterization tests protegidos | ✅ N/A — no se toca `test_qr_token.py` ni `test_table_sessions.py`; ningún test `"CONGELA comportamiento actual:"` se modifica. |
| IV. Nuevos specs pueden introducir nuevo comportamiento | ✅ Cumple — la resiliencia ante recarga y la vista de pasos son el comportamiento nuevo que autoriza spec 034. |
| V. Sin refactors oportunistas | ✅ Cumple — el reemplazo de iconos se acota explícitamente a esta vista (spec 034, Fuera de Alcance); no se toca iconografía de otras pantallas del comensal, cocina o caja. |
| VI. Evolución incremental | ⚠️ **Atención** — la spec agrupa 4 clases de cambio (resiliencia/nueva capacidad, arquitectura de UI modal→vista, corrección de presentación de datos, reemplazo de iconos). Se mitiga porque la propia spec ya las separa en 4 historias de usuario independientemente probables y priorizadas (P1/P1/P2/P3, Historias 1-4) — `/speckit-tasks` DEBE mantener esa separación como unidades verificables independientes, no como una sola entrega monolítica. |
| VII. Compatibilidad con datos históricos | ✅ N/A — no se toca facturación ni ventas históricas. |
| VIII. Evolución del modelo de datos | ✅ Cumple, sin migración — el campo `format` ya existe en `PaymentMethodCatalog.fields` (spec 032); esta funcionalidad solo lo agrega a una respuesta que el comensal ya consulta (`GET /cart/payment-methods`), sin tabla ni columna nueva. |
| IX. Dependencias nuevas justificadas | ✅ N/A — no se agrega ninguna dependencia nueva (se reutilizan `app-icon`, `localStorage`, y los esquemas Pydantic ya existentes). |
| X. Verificación obligatoria | ✅ Cumple — se verifica con los scripts autoejecutables del backend (uno nuevo o ampliado para el campo de respuesta agregado) y specs Vitest del frontend, contra los criterios de aceptación del spec 034. |
| XI. Negocio vs técnico | ✅ Cumple — las dos decisiones de negocio (alcance por dispositivo del progreso recuperable; criterio objetivo de icono) ya están registradas en spec 034 (Clarifications, sesión 2026-08-24). |
| XII. Trazabilidad | ✅ Cumple — spec 034 → este plan → research/data-model/quickstart → tasks (siguiente comando) → implementación. |
| XIII. Español de Colombia | ✅ Cumple — spec, plan, research, data-model y quickstart de esta funcionalidad se escriben en español de Colombia. |

**Resultado**: ninguna violación bloqueante. El único punto de atención (Principio VI) se resuelve
manteniendo la separación por historia de usuario ya definida en el spec durante `/speckit-tasks`,
no mediante una excepción constitucional.

## Project Structure

### Documentation (this feature)

```text
specs/034-checkout-qr-vista-pasos/
├── plan.md              # Este archivo (/speckit-plan)
├── research.md          # Fase 0 (/speckit-plan)
├── data-model.md         # Fase 1 (/speckit-plan)
├── quickstart.md         # Fase 1 (/speckit-plan)
├── contracts/
│   └── cart-payment-methods.md   # Fase 1 (/speckit-plan)
└── tasks.md              # Fase 2 (/speckit-tasks — NO se crea aquí)
```

### Código fuente (fuera de este repo de specs)

```text
# Backend — /home/deimer/Documents/projects/inventario/pos-backend
app/api/v1/cart/
├── router.py     # GET /cart/payment-methods (respuesta ampliada con `fields`)
├── service.py     # list_payment_methods: junta PaymentMethodCatalog.fields con la config del tenant
└── schemas.py     # DinerPaymentMethod: nuevo campo `fields: list[PaymentMethodFieldDefinition]`

# Frontend — /home/deimer/Documents/projects/inventario/pos-heladeria
src/app/modules/tables/pages/
└── public-menu.component.ts        # se retira el modal de revisión/pago (reviewOpen, reviewStep, ...)

src/app/modules/tables/pages/checkout/     # NUEVO — vista de pasos dedicada
├── checkout-progress.store.ts             # NUEVO — lee/escribe el progreso en localStorage
├── review-step.component.ts               # resumen del pedido
├── payment-method-step.component.ts        # selección de método
├── transfer-details-step.component.ts      # datos de pago + carga de comprobante
└── confirmation-step.component.ts          # confirmación (pedido ya creado)

src/app/shared/icon/icon.component.ts   # nuevos @case (transferencia, adjuntar/quitar, volver, confirmar)
src/app/app.routes.ts                    # nueva ruta hija/hermana bajo menu/t/:token para el checkout
```

**Structure Decision**: Aplicación web ya existente (frontend + backend, ambos en producción); esta
funcionalidad no agrega repositorios ni servicios nuevos. El cambio se concentra en `pos-heladeria`
(nueva vista de pasos + almacén de progreso local) y un ajuste puntual de respuesta en
`pos-backend` (agregar metadata de formato de campo a `GET /cart/payment-methods`) — sin nueva base
de datos ni nueva infraestructura de sesión.

## Complexity Tracking

> No se registran violaciones que requieran justificación. El único punto de atención detectado
> (Principio VI, Evolución Incremental) se gestiona con la descomposición por historia de usuario ya
> definida en el spec (ver Constitution Check arriba), no con una excepción constitucional.
