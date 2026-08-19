# Implementation Plan: Rediseño UX de Confirmación de Pago y Comanda en Terminal de Mesas (Skeilopos)

**Branch**: `026-mejoras-ux-comanda` | **Date**: 2026-08-18 | **Spec**: [spec.md](./spec.md)

**Input**: Feature specification from `/specs/026-mejoras-ux-comanda/spec.md`

## Summary

Hoy `confirm_cash_payment_attempt`/`approve_payment_attempt` (`app/api/v1/orders/checkout.py`)
solo cambian el estado del intento de pago; el envío a cocina (descuento de inventario,
`recibida → abierta`) sigue siendo una llamada manual aparte a `confirm_order` — el botón
"Confirmar" de `pending-orders-panel.component.ts`, deshabilitado hasta que el pago está
confirmado. Esta spec fusiona ambos pasos en una sola transacción atómica (FR-001/FR-002): se
extrae la lógica de `confirm_order` (sin su propio `commit`/`rollback`) a una función interna que
las dos funciones de confirmación de pago invocan antes de su propio `commit`, de forma que un
fallo de inventario revierte también la confirmación del pago (nunca queda "pagado sin cocina").
Además corrige un defecto de presentación ya localizado en el código: `OrderPaymentAttempt.
change_amount` ya se calcula y se serializa hoy (`schemas.py:195-196`), pero
`payment-attempt-review-panel.component.ts` solo lo muestra en un toast efímero (línea ~200) — la
vista persistente del intento confirmado (línea ~111-112) no lo incluye (FR-004/FR-005). Por
último expone división de cuenta y cobro combinado en el panel "Cuenta de la mesa" — capacidad que
`SessionBillPanelComponent`/`SplitBillPanelComponent` ya implementan casi por completo reutilizando
`compute_bill`/`close_session` (spec 010) y `build_sale` (spec 011) — y sube el tamaño de texto y
de los controles táctiles de los tres paneles de la Terminal de Mesas a un mínimo medible (FR-010/
FR-011).

## Technical Context

**Language/Version**: Backend — Python 3.14 (venv `pos-backend/env`). Frontend — TypeScript 5.9.2
(Angular 21.1.x, standalone components + signals, sin NgModules).

**Primary Dependencies**:
- Backend: FastAPI, SQLAlchemy 2.0 (sync), Pydantic 2. Ninguna dependencia nueva — esta spec no
  agrega tablas, columnas ni endpoints, solo reordena lógica ya existente entre funciones ya
  existentes de `checkout.py` (Principio IX: no aplica justificación porque no se añade nada).
- Frontend: Angular 21 (standalone + signals), Tailwind CSS v4 (sin Angular Material — estilo
  propio a base de utilidades Tailwind, iconos emoji, paleta ámbar/esmeralda/rojo/índigo ya en uso).
  Ninguna dependencia nueva.

**Storage**: PostgreSQL 16 schema-per-tenant. **Sin migraciones** — `OrderPaymentAttempt.
amount_received`/`change_amount` ya existen como columnas (`app/models/order_payment_attempt.py:
41-42`) y ya se serializan en `PaymentAttemptResponse` (`schemas.py:195-196`); esta spec es
enteramente de comportamiento (backend) y presentación (frontend) sobre datos que ya existen.

**Testing**: Backend — `unittest` vía `python -m unittest` (sin pytest en el repo). Se extiende
`app/characterization_tests/test_orders_checkout.py` (el CONGELA de `confirm_order`, línea 288,
citando la autorización de esta spec para el cambio de FR-017) y
`test_orders_payment_gate.py` (no es CONGELA, libre — spec 024 ya lo usa para FR-017/FR-018).
Frontend — Vitest + `@angular/build:unit-test`, specs colocados (`*.spec.ts`), mismo patrón que
`session-bill-panel.component.spec.ts` (que ya protege la regresión del placeholder "Selecciona una
mesa con consumo").

**Target Platform**: Linux server (API `pos-backend`) + navegador (SPA `pos-heladeria`), usado por
el personal (cajero/mesero) tanto en tablet táctil como en escritorio con mouse/teclado (spec,
Clarifications).

**Project Type**: Web application (backend FastAPI + frontend Angular, dos repositorios
independientes, siblings de este repo `pos-specs`).

**Performance Goals**: sin objetivo de throughput nuevo. La fusión de FR-001 no agrega ninguna
escritura nueva a la base de datos — mueve una escritura que ya ocurría (`deduct_order_items` +
`order.status = "abierta"`) a la misma transacción que otra escritura que ya ocurría (confirmar el
intento de pago), sin agregar round-trips HTTP nuevos: la respuesta síncrona de `confirm-cash`/
`approve` ya es la señal de "de inmediato" que pide FR-004 (research.md, Decisión 4).

**Constraints**:
- No se agrega ningún endpoint nuevo ni se cambia el shape de request/response de
  `confirm-cash`/`approve`/`reject` — solo su efecto secundario (FR-001/FR-002).
- `POST /orders/{id}/confirm` (`confirm_order`) se mantiene expuesto sin cambios de contrato, como
  vía de recuperación manual — no se elimina (research.md, Decisión 2).
- No se agrega ningún estado nuevo a `CustomerOrder.status` ni al `CHECK` que lo protege (mismo
  invariante que ya citó spec 024).
- FR-010/FR-011 fijan mínimos concretos (texto ≥16px equivalente en escritorio, controles táctiles
  ≥44×44pt) — ya resueltos en Clarifications, no quedan como "NEEDS CLARIFICATION" técnicos.
- Fuera de alcance explícito de `spec.md`: nuevos métodos de pago, reabrir la prohibición de
  división porcentual, cambios al motor de cálculo de factura/promociones, mockups visuales
  concretos (se resuelven como parte de las tareas de implementación, no de este plan).

**Scale/Scope**: 0 tablas/columnas nuevas, 0 endpoints nuevos; 1 función interna nueva en
`checkout.py` (extracción sin cambio de comportamiento observable del endpoint existente), 2
funciones modificadas (`confirm_cash_payment_attempt`, `approve_payment_attempt`) para invocarla; en
`pos-heladeria`, ~5 componentes ya existentes de `modules/tables/` modificados (ninguno nuevo) —
`pending-orders-panel`, `payment-attempt-review-panel`, `pos-tables-panel`, `session-bill-panel`,
`split-bill-panel`.

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| Principio | Evaluación | Estado |
|---|---|---|
| **I. Las Nuevas Funcionalidades Nacen de un Spec** | `spec.md` existe, aprobado, con 4 historias priorizadas, 12 FRs y 5 clarificaciones ya resueltas (sesiones `/speckit-specify` y `/speckit-clarify`, 2026-08-18) antes de este plan. | PASS |
| **II. El Comportamiento Existente Sigue Protegido** | El único comportamiento existente que cambia es la precondición/secuencia de `confirm_order` (spec 024, FR-017: pasa de "acción manual separada" a "disparada automáticamente al confirmar el pago"). Es una decisión de negocio explícita, documentada en las Clarifications del propio `spec.md` (quién: usuario, vía sesión interactiva; cuándo: 2026-08-18; qué cambia: FR-017; por qué: la separación generaba la percepción de que el pedido "no pasó a la mesa"; qué se ve afectado: `confirm_cash_payment_attempt`, `approve_payment_attempt`, `pending-orders-panel.component.ts`) — mismo criterio que spec 024 §Constitution Check aplicó a su propio cambio de `confirm_order`: al ser comportamiento nuevo autorizado por un spec de evolución funcional (Principio IV), no exige una entrada aparte en `registro-de-anomalias.md` (ese registro es el libro de anomalías heredadas de la fase de reconocimiento, no el lugar donde se documentan las decisiones de negocio de specs nuevas, que se autodocumentan en su propia sección de Clarifications). Ningún otro comportamiento de `checkout.py`/`table_sessions` cambia — `confirm_order` en sí (su propio contrato como endpoint) permanece intacto. | PASS |
| **III. Los Characterization Tests Protegen el Comportamiento Heredado** | `test_orders_checkout.py:288` (CONGELA sobre `confirm_order:307-355`) se actualiza para reflejar el refactor a `_confirm_order_impl` — comportamiento observable del endpoint público sin cambios, pero la cita de líneas y la forma de invocarlo internamente sí cambian, así que el test se toca citando esta spec (FR-001) como autorización, con evidencia de que el resto de casos protegidos (confirmación directa vía endpoint, precondición de pago, bloqueo por falta de stock) sigue en verde. Ningún test CONGELA de `test_table_sessions_service.py`/`test_table_sessions_split_blindaje.py` se toca — US3 reutiliza `compute_bill`/`close_session`/`build_sale` sin modificarlos. | PASS (condicionado a verificar en la fase de implementación, no en este plan) |
| **IV. Los Nuevos Specs Pueden Introducir Nuevo Comportamiento** | El cambio de FR-001/FR-002 es comportamiento nuevo explícito; el resto de la spec (FR-003 a FR-012) es presentación sobre datos y reglas ya existentes — no se exige equivalencia con el pasado, solo conformidad con `spec.md`. | PASS |
| **V. Nuevas Funcionalidades Antes que Refactorizaciones Oportunistas** | `SessionBillPanelComponent`/`SplitBillPanelComponent` (división de cuenta, multi-pago) ya funcionan — esta spec los audita contra los acceptance scenarios de US3 y solo ajusta lo que realmente falte (research.md, Decisión 7), sin reescribirlos. `TablesPageComponent` (CRUD de mesas/QR, pantalla distinta) no se toca. | PASS |
| **VI. Evolución Incremental** | El alcance se divide en las mismas unidades que las historias del spec (US1 fusión pago→cocina → US2 mostrar cambio → US3 auditoría+ajuste de cuenta de mesa → US4 tamaños/legibilidad transversal), cada una verificable por separado. No se mezcla con ninguna migración de datos ni cambio de arquitectura. | PASS |
| **VII. Compatibilidad con Datos Históricos** | No se toca `Sale`/`Payment`/`SaleInvoice` ni ninguna venta ya facturada — `build_sale` se reutiliza sin modificar su lógica de emisión. | PASS |
| **VIII. Evolución del Modelo de Datos** | Sin cambios de esquema — ver Technical Context (Storage) y data-model.md. | PASS (no aplica) |
| **IX. Dependencias Nuevas Permitidas con Justificación** | No se añade ninguna dependencia. | PASS (no aplica) |
| **X. Verificación Obligatoria** | Cada historia de usuario tiene su "Independent Test" en `spec.md`; quickstart.md los traduce a comandos `unittest`/Vitest ejecutables, más los characterization tests existentes de `orders`/`table_sessions` como red de no-regresión. | PASS |
| **XI. Decisiones de Negocio Frente a Decisiones Técnicas** | La fusión pago→cocina es la decisión de negocio (spec, Clarifications); *cómo* lograr la atomicidad (extraer `_confirm_order_impl` sin `commit` propio) es la decisión técnica correspondiente, documentada en research.md sin mezclarse con la de negocio. | PASS |
| **XII. Trazabilidad** | Cadena completa: `spec.md` (Necesidad+Spec+Decisión, incluida la sesión de clarificación) → este `plan.md`/`research.md` (Decisión técnica) → tareas de `tasks.md` (Fase 2, no generada por este comando) → implementación → characterization tests + tests nuevos → `quickstart.md` (Verificación). | PASS |
| **XIII. Todo en Español de Colombia** | Este plan y todos sus artefactos (research.md, data-model.md, contracts/, quickstart.md) se escriben en español de Colombia, igual que `spec.md`. | PASS |

Sin violaciones. La tabla de Complexity Tracking al final de este documento queda vacía.

## Project Structure

### Documentation (this feature)

```text
specs/026-mejoras-ux-comanda/
├── plan.md              # Este fichero (/speckit-plan)
├── research.md          # Fase 0 (/speckit-plan) — decisiones técnicas y alternativas descartadas
├── data-model.md         # Fase 1 (/speckit-plan) — entidades reutilizadas, sin cambios de esquema
├── quickstart.md         # Fase 1 (/speckit-plan) — validación ejecutable por historia de usuario
├── contracts/            # Fase 1 (/speckit-plan) — contratos HTTP modificados (efecto, no forma)
│   ├── payment-confirmation.md
│   ├── order-confirm-manual.md
│   └── table-bill-multi-payment.md
└── tasks.md              # Fase 2 (/speckit-tasks — NO se crea en este comando)
```

### Source Code (repositorios sibling de `pos-specs`)

Esta spec vive en `pos-specs`, pero el código que describe está en `../pos-backend` y
`../pos-heladeria` (Constitución §Alcance). Rutas relativas a la raíz de cada repo.

```text
# pos-backend
app/api/v1/orders/
├── checkout.py                    # MODIFICADO — se extrae _confirm_order_impl(db, order_id,
│                                     user) de confirm_order (misma lógica, sin su propio commit/
│                                     rollback); confirm_cash_payment_attempt y
│                                     approve_payment_attempt la invocan antes de su propio commit
│                                     (FR-001/FR-002); confirm_order pasa a ser un wrapper delgado
│                                     que llama _confirm_order_impl + commit/rollback (mismo
│                                     contrato observable, endpoint sin cambios)
└── router.py                       # SIN CAMBIOS — mismos 5 endpoints, mismo shape

app/characterization_tests/
├── test_orders_checkout.py         # MODIFICADO — CONGELA de confirm_order (línea 288) actualizado
│                                      citando la autorización de spec 026 FR-001; se agregan
│                                      aserciones para el efecto combinado en confirm-cash/approve
└── test_orders_payment_gate.py     # MODIFICADO (no es CONGELA) — nuevos casos: éxito combinado
                                       (US1), fallo de stock deja el intento sin confirmar (US1
                                       escenario 3), doble confirmación no duplica el envío a cocina

# pos-heladeria
src/app/modules/tables/components/
├── pending-orders-panel.component.ts        # MODIFICADO — se elimina el botón "Confirmar", el
│                                               método confirm() y isPaymentConfirmed() (FR-001);
│                                               se conserva reject()/cancelOrder (recuperación,
│                                               FR-002); se revisa el texto/badge de la pestaña
│                                               ("Por confirmar" ya no describe con precisión lo
│                                               que queda: comprobantes pendientes de revisión)
├── payment-attempt-review-panel.component.ts # MODIFICADO — el bloque "✓ Pago confirmado" (línea
│                                               ~111-112) pasa a mostrar también monto recibido y
│                                               cambio de forma permanente (FR-004/FR-005),
│                                               reutilizando los campos que `PaymentAttempt` ya
│                                               expone; el toast (línea ~200-202) se conserva como
│                                               confirmación inmediata adicional, no como único lugar
├── pos-tables-panel.component.ts             # MODIFICADO — ajuste de tamaño de texto/controles
│                                               (FR-010/FR-011) y verificación de que cada estado
│                                               lleve etiqueta/ícono, no solo color (FR-003)
├── session-bill-panel.component.ts           # MODIFICADO — auditoría contra US3 (research.md,
│                                               Decisión 7) + ajuste de tamaño (FR-010/FR-011)
└── split-bill-panel.component.ts             # MODIFICADO — auditoría contra US3 + ajuste de
                                                tamaño (FR-010/FR-011); sin cambios a la regla de
                                                reparto por ítem/unidad (`AssignRow.units[]`)

src/app/modules/tables/services/
├── dining-session.service.ts        # SIN CAMBIOS DE CONTRATO — mismos métodos/endpoints
└── pos-terminal.store.ts            # MODIFICADO si el badge de "Por confirmar" cambia de contar
                                        "pendientes de enviar a cocina" a "comprobantes pendientes
                                        de revisión" (consecuencia directa de FR-001)

src/app/modules/tables/pages/
└── table-sessions.component.ts      # MODIFICADO solo si el rótulo de la pestaña central cambia
                                        (consecuencia de FR-001, no un rediseño de la pestaña en sí)
```

**Structure Decision**: cada historia de usuario del spec se mapea a un subconjunto disjunto de los
ficheros de arriba (US1 → `checkout.py` + `pending-orders-panel`; US2 →
`payment-attempt-review-panel`; US3 → `session-bill-panel` + `split-bill-panel` (auditoría); US4 →
los cinco componentes de `modules/tables/components/`, ajuste transversal de tamaño), consistente
con Principio VI. No se crea ningún componente, servicio, módulo ni endpoint nuevo — toda la
funcionalidad vive en ficheros que ya existen y ya implementan la mayor parte del comportamiento
que pide el spec (research.md).

## Complexity Tracking

> **Fill ONLY if Constitution Check has violations that must be justified**

Sin violaciones — tabla vacía.
