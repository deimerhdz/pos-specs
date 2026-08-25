# Implementation Plan: Estado "Pagada" Correcto y Formato de Moneda Reutilizable

**Branch**: `035-estado-pagado-formato-moneda` | **Date**: 2026-08-25 | **Spec**: [spec.md](./spec.md)

**Input**: Especificación de la funcionalidad en `specs/035-estado-pagado-formato-moneda/spec.md`

## Summary

Dos correcciones independientes (Principio VI):

1. **Estado de pago correcto**: `checkout_and_send` (el único camino de cobro que hoy deja
   `CustomerOrder.status = 'abierta'` aunque ya exista una `Sale`) pasa a fijar
   `status = 'pagada'` en la misma transacción. Las tres funciones de
   `orders/tables_advanced.py` que hoy usan `status` para decidir si una mesa "todavía tiene
   trabajo pendiente" (liberar, mover, fusionar) dejan de tratar `'pagada'` como sinónimo de
   "sin nada pendiente" — pasan a mirar si a la orden le quedan ítems sin terminar de
   preparar (`estado_cocina` distinto de `listo`/`anulado`).
2. **Input de moneda reutilizable**: un componente Angular nuevo (`ControlValueAccessor`,
   mismo patrón que `shared/password-input/`) que formatea con separador de miles (punto,
   convención ya usada por `shared/money.ts`) mientras se escribe, entregando siempre un
   número limpio al formulario. Se aplica a los ~12 campos de precio/monto ya identificados.

Sin dependencias nuevas (Principio IX n/a), sin migración de base de datos (los campos
`CustomerOrder.status`/`OrderItem.estado_cocina` ya existen).

## Technical Context

**Language/Version**: Backend: Python 3.14 (FastAPI 0.136.3), sin cambio de versión.
Frontend: TypeScript 5.9 / Angular 21.1 (componentes standalone, signals), sin cambio de
versión.

**Primary Dependencies**: Backend: SQLAlchemy — solo modelos y consultas ya existentes
(`CustomerOrder`, `OrderItem`, `Sale`); ninguna dependencia nueva. Frontend: Angular Forms
(`ControlValueAccessor`, mismo patrón de auto-inyección de `NgControl` ya usado en
`shared/password-input/password-input.component.ts` y
`shared/searchable-select/searchable-select.component.ts`) e `Intl.NumberFormat`/
`toLocaleString('es-CO')` nativos — sin librería de máscaras/moneda nueva (no hay ninguna
instalada hoy en `pos-heladeria`, confirmado en investigación previa a esta spec).

**Storage**: PostgreSQL 16, schema-per-tenant. Sin migración: `CustomerOrder.status`
(`ck_customer_order_status`, ya admite `'pagada'` en el CHECK) y
`OrderItem.estado_cocina` ya existen; esta spec solo agrega un punto adicional del ciclo de
vida donde `status` toma el valor `'pagada'` y cambia la condición SQL de tres consultas
existentes.

**Testing**: Backend: `python -m unittest app.characterization_tests.*` (characterization
tests, algunos protegidos con `"CONGELA comportamiento actual:"`) + scripts autoejecutables
bajo `app/scripts/test_*.py` (Principio X). Frontend: Vitest vía `@angular/build:unit-test`
(`ng test`) — nota: al momento de este plan, `ng test` falla en la compilación por errores
de TypeScript preexistentes y no relacionados en specs de otro módulo (`reports`,
`super-admin/tenant.service.spec.ts`, `tenant-date.pipe.spec.ts`, todos referenciando un
campo `plan` que no pertenece a esta funcionalidad); no se corrigen aquí (Principio V — no es
una refactorización relacionada), se deja constancia para que no se confunda con una
regresión introducida por esta spec.

**Target Platform**: Web — Terminal de Mesas y listado de Órdenes (staff dashboard) para la
Historia 1; prácticamente todos los módulos del dashboard de staff (Caja, Productos,
Inventario, Promociones, Mesas, Super Admin) para la Historia 2 — ambos ya en producción.

**Project Type**: Aplicación web ya existente (frontend + backend, dos repos en producción);
esta funcionalidad no agrega ningún repositorio ni servicio nuevo.

**Performance Goals**: Sin metas nuevas. Historia 1 no agrega ninguna consulta ni round-trip
extra: el cambio de `status` viaja en la misma transacción ya abierta de
`checkout_and_send`, y `_active_orders_on_table` pasa de un `WHERE status NOT IN (...)` a
cargar las órdenes de la mesa con sus ítems (`selectinload`, una sola consulta adicional por
`order_id`, ya indexado por FK) y filtrar en Python — mismo patrón que el resto del módulo ya
usa para `estado_cocina` (`research.md`, Decisión 2), sin impacto perceptible dado el volumen
de órdenes por mesa. Historia 2 formatea en cada pulsación de tecla — debe sentirse
instantáneo (sin lag perceptible) en el hardware ya usado por el staff.

**Constraints**: Historia 1 no debe cambiar el significado de `paid`/`order_has_sale`
(spec 029) ni las protecciones que ya usan esa señal correctamente (anular un ítem de un
pedido ya pagado, listado de Órdenes) — el cambio se acota exactamente a las tres funciones
de `tables_advanced.py` identificadas en FR-003, y a `checkout_and_send`. Historia 2 debe
preservar "vacío se queda vacío" (FR-009) y soporte de centavos donde ya existe (FR-010), y
solo aplicar el formato quando un campo dual porcentaje/pesos está en modo pesos (FR-011).

**Scale/Scope**: Historia 1 — 1 cambio de comportamiento en `checkout.checkout_and_send` +
3 funciones ajustadas en `orders/tables_advanced.py` (`_active_orders_on_table`, y los
chequeos puntuales de `move_order`/`merge_orders`), todo en `pos-backend`. Historia 2 — 1
componente nuevo en `pos-heladeria/src/app/shared/money-input/` + ~12 puntos de uso
migrados en 12 archivos ya identificados (ver `spec.md`, Assumptions).

## Constitution Check

*GATE: Debe pasar antes de la Fase 0. Se re-evalúa después del Diseño de Fase 1.*

| Principio | Evaluación |
|---|---|
| I. Nace de un spec | ✅ Cumple — spec 035 aprobada (con clarificaciones resueltas) antes de este plan. |
| II. Comportamiento existente protegido | ⚠️ **Atención, ya autorizado por esta spec** — Historia 1 revierte, para el camino puntual de `checkout_and_send`, la decisión de spec 029 de no tocar `status`. La spec (Clarifications + Assumptions) ya documenta qué cambia, por qué, y qué funcionalidades quedan afectadas (FR-003). Falta registrar esta decisión en `specs/000-reconocimiento/registro-de-anomalias.md` **antes** de implementar — se deja como tarea explícita en `quickstart.md`. |
| III. Characterization tests protegidos | ✅ Cumple, verificado — el único test `"CONGELA comportamiento actual:"` que ejercita la guarda de `set_table_status` (`test_orders_tables_advanced.py:54-69`) usa una orden **sin ítems**; con la nueva condición ("¿queda algún ítem sin terminar?"), un pedido sin ítems no bloquea, así que ese test sigue pasando sin modificarlo. Los CONGELA de `move_order`/`merge_orders` (líneas 73-125) tampoco tocan el caso `'pagada'` con ítems pendientes. El único test que sí debe actualizar su aserción es `test_checkout_and_send_cobra_descuenta_y_abre_a_cocina` (`test_orders_checkout.py:546-573`), cuyo docstring dice explícitamente "Comportamiento nuevo (spec 028)" — **no** es un test CONGELA, así que su actualización no requiere el procedimiento del Principio III, solo reflejar el nuevo comportamiento autorizado por esta spec. |
| IV. Nuevos specs pueden introducir nuevo comportamiento | ✅ Cumple — el estado `'pagada'` correcto y el input de moneda son el comportamiento nuevo que autoriza spec 035. |
| V. Sin refactors oportunistas | ✅ Cumple — la migración de Historia 2 se acota a los ~12 campos ya identificados en `spec.md`; no se toca ningún otro input ni se corrige la duplicación de `formatMoney` ya detectada en `cash-session.store.ts:569` (fuera de alcance, mencionada solo como nota). |
| VI. Evolución incremental | ✅ Cumple — dos historias independientes y desplegables por separado; dentro de Historia 1, el cambio se acota a un único camino de cobro (`checkout_and_send`) sin tocar `pay_order`/`close_session` (ya correctos) ni el camino QR (`_confirm_order_impl`, que no debe cambiar, FR-002). |
| VII. Compatibilidad con datos históricos | ✅ N/A — no se recalcula ninguna venta ni factura ya emitida; los pedidos cobrados antes de este cambio conservan su `status` histórico tal cual (Edge Case de `spec.md`). |
| VIII. Evolución del modelo de datos | ✅ Cumple, sin migración — `CustomerOrder.status` ya admite `'pagada'` en su `CHECK` constraint; `OrderItem.estado_cocina` ya existe. Esta spec solo agrega un punto adicional del ciclo de vida donde se escribe un valor ya válido, y cambia la condición de lectura de tres consultas. |
| IX. Dependencias nuevas justificadas | ✅ N/A — no se agrega ninguna dependencia; se reutiliza el patrón CVA ya establecido (`shared/password-input/`) y `Intl.NumberFormat`/`toLocaleString` nativos. |
| X. Verificación obligatoria | ✅ Cumple — se verifica con los characterization tests existentes (sin modificar los que no deben cambiar), un test nuevo para el caso divergente (`'pagada'` con ítems pendientes), la actualización explícita del test no-CONGELA de `checkout_and_send`, y los criterios de aceptación de `spec.md`. |
| XI. Negocio vs técnico | ✅ Cumple — la decisión de negocio (revertir el `status` congelado de spec 029 para este camino puntual) y el separador de miles ya quedaron registrados en spec 035 (Clarifications, sesión 2026-08-25). |
| XII. Trazabilidad | ✅ Cumple — spec 035 → este plan → research/data-model/quickstart → tasks (siguiente comando) → implementación, incluyendo la entrada pendiente en `registro-de-anomalias.md` (Principio II). |
| XIII. Español de Colombia | ✅ Cumple — spec, plan, research, data-model y quickstart de esta funcionalidad se escriben en español de Colombia. |

**Resultado**: ninguna violación bloqueante. El único punto de atención (Principio II) ya
está autorizado por la propia spec 035 — falta únicamente el registro formal en
`registro-de-anomalias.md`, que se deja como paso explícito antes de implementar (ver
`quickstart.md`).

## Project Structure

### Documentation (this feature)

```text
specs/035-estado-pagado-formato-moneda/
├── plan.md              # Este archivo (/speckit-plan)
├── research.md          # Fase 0 (/speckit-plan)
├── data-model.md         # Fase 1 (/speckit-plan)
├── quickstart.md         # Fase 1 (/speckit-plan)
├── contracts/
│   ├── checkout-and-send-status.md    # Fase 1 (/speckit-plan)
│   └── money-input-component.md       # Fase 1 (/speckit-plan)
└── tasks.md              # Fase 2 (/speckit-tasks — NO se crea aquí)
```

### Código fuente (fuera de este repo de specs)

```text
# Backend — /home/deimer/Documents/projects/inventario/pos-backend
app/api/v1/orders/
├── checkout.py           # checkout_and_send: fija status = 'pagada' tras crear la Sale
└── tables_advanced.py    # _active_orders_on_table, move_order, merge_orders: la
                           # condición de "trabajo pendiente" pasa a mirar estado_cocina

app/characterization_tests/
└── test_orders_checkout.py   # test_checkout_and_send_cobra_descuenta_y_abre_a_cocina:
                               # aserción actualizada a status == 'pagada' (no es CONGELA)
                               # + test nuevo para el caso 'pagada' con ítems pendientes
                               # (tables_advanced)

specs/000-reconocimiento/
└── registro-de-anomalias.md   # nueva entrada: decisión de negocio que autoriza revertir,
                                # para checkout_and_send, la decisión de spec 029

# Frontend — /home/deimer/Documents/projects/inventario/pos-heladeria
src/app/shared/money-input/
└── money-input.component.ts   # NUEVO — control CVA reutilizable, formato $ 50.000 en vivo

# ~12 puntos de uso migrados (ver spec.md Assumptions para la lista completa), p. ej.:
src/app/modules/cash-register/components/cash-open.component.ts
src/app/modules/cash-register/components/cash-movement-modal.component.ts
src/app/modules/cash-register/components/cash-dashboard.component.ts
src/app/modules/products/pages/product-form.component.ts
src/app/modules/option-groups/components/option-form.component.ts
src/app/modules/inventory/components/inventory-item-form.component.ts
src/app/modules/inventory/components/purchase-form.component.ts
src/app/modules/promotions/pages/promotions-page.component.ts
src/app/modules/promotions/components/scope-picker.component.ts
src/app/modules/tables/components/payment-input.component.ts
src/app/modules/tables/components/payment-attempt-review-panel.component.ts
src/app/modules/super-admin/components/plan-form.component.ts
```

**Structure Decision**: Aplicación web ya existente (frontend + backend, ambos en
producción); esta funcionalidad no agrega repositorios ni servicios nuevos. Historia 1 se
concentra en `pos-backend/app/api/v1/orders/` (dos archivos, sin nueva tabla ni endpoint).
Historia 2 agrega un componente compartido en `pos-heladeria/src/app/shared/` y migra los
puntos de uso ya identificados, sin nueva infraestructura ni dependencia.

## Complexity Tracking

> No se registran violaciones que requieran justificación. El único punto de atención
> (Principio II) se resuelve con el registro de decisión de negocio ya previsto en el flujo
> normal de esta fase (`registro-de-anomalias.md`), no con una excepción constitucional.
