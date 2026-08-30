# Implementation Plan: Carga diferida de datos y tarjetas de pedido de Domicilio/Para Llevar en la Terminal de Mesas

**Branch**: `059-terminal-mesas-carga-y-pedidos` | **Date**: 2026-08-29 | **Spec**: [spec.md](./spec.md)

**Input**: Feature specification from `/specs/059-terminal-mesas-carga-y-pedidos/spec.md`

**Note**: This template is filled in by the `/speckit-plan` command; its definition describes the execution workflow.

## Summary

La Terminal de Mesas (`table-sessions.component.ts`, `pos-heladeria`) hoy dispara, en un solo
`Promise.all` dentro de `PosTerminalStore.init()`, todas las peticiones de arranque sin distinguir
cuáles son necesarias para el primer render (mesas, pedidos, menú/catálogo, promociones) de cuáles
solo se usan dentro del panel de cobro (métodos de pago, turno de caja). Este plan (1) difiere esas
dos últimas peticiones hasta que el cajero selecciona un pedido real por primera vez en la sesión de
la app (`store.selectedOrder()` con valor), y (2) cierra la brecha por la que los pedidos
`TAKEAWAY`/`DELIVERY` ya creables desde `manual-order-page.component.ts` (spec 055/056) quedan
invisibles e inalcanzables en la Terminal de Mesas: las pestañas "Domicilios"/"Para llevar" pasan de
mostrar siempre el mensaje vacío fijo a filtrar `store.orders()` (que ya trae esos pedidos, sin
cambio de backend) por `order_type` y mostrarlos como tarjetas con el mismo formato visual que las
tarjetas de mesa; seleccionar una de esas tarjetas abre su detalle y su cobro en los paneles
central/derecho con el mismo comportamiento ya implementado para una mesa. El enfoque técnico:
extraer el markup de tarjeta ya inline en `pos-tables-panel.component.ts` a un componente
presentacional reutilizable, introducir un modelo de selección basado en "pedido seleccionado"
(hoy solo existe "mesa seleccionada") del que derivar `centralState()`/`hasActiveOrder()`, y mover
las dos llamadas diferidas a un punto disparado por esa misma selección.

## Technical Context

**Language/Version**: TypeScript 5.9.2, Angular 21.1 (standalone components, signals, nuevo
control-flow `@if`/`@for`/`@switch`) — mismo stack que el resto de `pos-heladeria`, sin cambios.

**Primary Dependencies**: Angular core/router/forms/cdk (`^21.1.x`), RxJS (interop vía
`firstValueFrom` en los `*.service.ts` de transporte HTTP) — ninguna dependencia nueva (Principio
IX de la Constitución: no aplica, este plan no introduce ninguna).

**Storage**: N/A — frontend puro; consume la API de `pos-backend` ya existente (`/orders`,
`/orders/tables`, `/sales/payment-methods`, `/cash-shifts/...`) sin ningún endpoint nuevo (spec,
Out of Scope).

**Testing**: Vitest (`@angular/build:unit-test`), mismo patrón de `*.spec.ts` colocados junto a
cada componente/servicio ya usado en el módulo (`pos-terminal.store.spec.ts`,
`pos-tables-panel.component.spec.ts`, `pos-checkout-panel.component.spec.ts`,
`pos-order-panel.component.spec.ts`, `table-sessions.component.spec.ts`). No se detectaron tests
con prefijo `"CONGELA comportamiento actual:"` dentro de `src/app/modules/tables/` (verificado por
grep) — no hay characterization tests que este cambio deba proteger explícitamente (Principio III
de la Constitución no aplica una restricción adicional aquí), pero los `*.spec.ts` ya existentes sí
verifican el comportamiento actual de selección de mesa y deben seguir pasando salvo los cambios
explícitamente descritos en el spec.

**Target Platform**: Navegador web de escritorio — terminal de punto de venta operada por el
personal de caja, misma SPA Angular ya en producción.

**Project Type**: Web application — cambio **exclusivamente de frontend** (`pos-heladeria`); el
spec descarta explícitamente cualquier cambio de `pos-backend` (ningún endpoint, campo ni
migración nueva — los datos que este plan necesita, `order_type`/`customer_name`/`delivery_*` en
`DiningOrder`, ya los expone la API desde spec 055/056).

**Performance Goals**: Eliminar 2 peticiones HTTP del arranque de la Terminal de Mesas
(`GET /sales/payment-methods` y `GET /sales/payment-methods?available=true`, más la resolución de
turno de caja) mientras no exista ningún pedido seleccionado — sin aumentar el tiempo hasta que la
grilla de mesas/pedidos esté operable (SC-001/SC-004 del spec).

**Constraints**: Ningún cambio de backend; el menú/catálogo y las promociones activas NO se
difieren (dependencia confirmada por código: `orderSubtotal()`, `pos-terminal.store.ts:1788`, los
usa para el total con descuento ya visible en cada tarjeta desde el primer render); el
comportamiento de cobro/checkout ya implementado para mesas (validación de pago QR, cálculo de
cambio, envío a cocina, facturación) no cambia, solo se hace alcanzable también para pedidos sin
mesa.

**Scale/Scope**: Un módulo (`src/app/modules/tables/`) de la SPA: 1 store
(`pos-terminal.store.ts`), 3 componentes existentes a modificar (`pos-tables-panel.component.ts`,
`pos-order-panel.component.ts`, `pos-checkout-panel.component.ts`) y 1 componente presentacional
nuevo (tarjeta reutilizable, extraída del markup inline existente).

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| Principio | Evaluación |
|---|---|
| I. Nace de un spec | ✅ `spec.md` aprobado (058→059, con sesión de Clarifications) precede este plan. |
| II. Comportamiento existente protegido | ✅ El spec documenta explícitamente los dos comportamientos que cambian (cuándo se piden ciertos datos; qué se muestra en pestañas hoy vacías) y por qué — no es un cambio de precio/inventario/facturación, así que **no aplica** entrada en `registro-de-anomalias.md` (mismo criterio que specs 045/048/049/055/056, todas correcciones de UI/navegación sin ese registro). |
| III. Characterization tests | ✅ N/A — no existen tests `"CONGELA comportamiento actual:"` en `src/app/modules/tables/` (verificado). Los `*.spec.ts` funcionales existentes deben seguir pasando salvo los casos que el spec cambia explícitamente. |
| IV. Nuevo comportamiento vía spec | ✅ El spec define el comportamiento nuevo (Historias 2/3) explícitamente, no exige equivalencia total con el estado anterior. |
| V. Sin refactors oportunistas | ✅ La extracción de la tarjeta a componente reutilizable **no es un refactor oportunista**: es requisito directo del spec (FR-005, "mismo formato visual… reutilizada"), no una mejora no relacionada. Ningún otro archivo del módulo se toca sin estar ligado a un FR. |
| VI. Evolución incremental | ✅ El plan separa explícitamente dos unidades verificables independientes (carga diferida vs. tarjetas+selección de pedido), reflejando las 3 historias de usuario priorizadas P1 del spec; no mezcla con ninguna migración de datos ni cambio de arquitectura de backend. |
| VII. Datos históricos | ✅ N/A — no se toca ningún cálculo de venta/factura ya emitida; el flujo de cobro reutilizado es el mismo ya implementado. |
| VIII. Evolución del modelo de datos | ✅ N/A — cero cambios de modelo de datos (frontend puro, ver Out of Scope del spec). |
| IX. Dependencias nuevas | ✅ N/A — ninguna dependencia nueva. |
| X. Verificación obligatoria | ✅ Cubierto por Phase 1 (quickstart.md) + tests unitarios de store/componentes actualizados junto a la implementación (spec 059, tasks futuras). |
| XI. Negocio vs. técnico | ✅ El spec ya distingue la decisión de negocio (mostrar/cobrar pedidos Domicilio/Para llevar desde esta pantalla) de las decisiones técnicas que resuelve este plan (dónde vive el estado de "pedido seleccionado sin mesa"). |
| XII. Trazabilidad | ✅ Cadena spec 059 → Clarifications → este plan → tasks.md (siguiente comando) → implementación. |
| XIII. Español de Colombia | ✅ Este plan y sus artefactos se escriben en español de Colombia, igual que el spec. |

**Resultado**: PASS. Sin violaciones que requieran `Complexity Tracking`.

**Re-check post-Phase 1**: el diseño resultante (research.md + data-model.md + contracts/) no
introduce ninguna entidad persistida, ningún endpoint, ninguna dependencia nueva, ni ningún test
`"CONGELA comportamiento actual:"` afectado — el único elemento nuevo de superficie (el componente
`OrderSummaryCardComponent`) es presentacional y está directamente atado a FR-005 del spec, no a
una refactorización oportunista (Principio V). PASS confirmado, sin cambios respecto a la
evaluación pre-Phase 0.

## Project Structure

### Documentation (this feature)

```text
specs/059-terminal-mesas-carga-y-pedidos/
├── plan.md              # This file (/speckit-plan command output)
├── research.md          # Phase 0 output (/speckit-plan command)
├── data-model.md        # Phase 1 output (/speckit-plan command)
├── quickstart.md        # Phase 1 output (/speckit-plan command)
├── contracts/           # Phase 1 output (/speckit-plan command)
│   └── ui-contracts.md
└── tasks.md             # Phase 2 output (/speckit-tasks command - NOT created by /speckit-plan)
```

### Source Code (repository root)

Este plan modifica **exclusivamente** `../pos-heladeria` (frontend). No toca `../pos-backend` — el
spec descarta explícitamente cualquier endpoint, campo o migración nueva (los datos ya existen
desde spec 055/056).

```text
pos-heladeria/
└── src/app/modules/
    ├── tables/
    │   ├── services/
    │   │   └── pos-terminal.store.ts            # store principal — MODIFICAR
    │   │       (defiere payment-methods/cash-shift; agrega selección de
    │   │        "pedido sin mesa"; filtra órdenes por order_type para las
    │   │        pestañas Domicilios/Para llevar)
    │   ├── components/
    │   │   ├── pos-tables-panel.component.ts     # MODIFICAR
    │   │   │   (extrae la tarjeta inline a un componente nuevo; renderiza
    │   │   │    tarjetas de pedido en las pestañas Domicilios/Para llevar
    │   │   │    en vez del mensaje vacío fijo)
    │   │   ├── order-summary-card.component.ts   # NUEVO — tarjeta
    │   │   │   presentacional reutilizada por mesas y por pedidos
    │   │   │   Domicilio/Para llevar (FR-005/FR-007)
    │   │   ├── pos-order-panel.component.ts      # MODIFICAR
    │   │   │   (deja de gatear el detalle únicamente en `selectedTableId`;
    │   │   │    también muestra el detalle de un pedido sin mesa)
    │   │   └── pos-checkout-panel.component.ts   # MODIFICAR
    │   │       (mueve el disparo de payment-methods/cash-shift a la
    │   │        selección de pedido; ya reutiliza el mismo flujo de cobro
    │   │        para un pedido sin mesa sin cambios adicionales)
    │   └── pages/
    │       └── table-sessions.component.ts       # sin cambios funcionales
    │           (sigue orquestando los mismos componentes hijos)
    └── sales/services/
        └── payment-method.service.ts             # sin cambios (ya expone
            el mismo `load()`/`loadAvailableForCheckout()` reutilizado, solo
            cambia desde dónde se invoca)
```

**Structure Decision**: Cambio contenido en un único módulo de la SPA
(`pos-heladeria/src/app/modules/tables/`), siguiendo la estructura de módulos por dominio ya
existente en el repositorio (no se introduce ninguna carpeta ni convención nueva). Un componente
presentacional nuevo (`order-summary-card.component.ts`) se agrega junto a los componentes
existentes de `tables/components/`, mismo patrón que ya siguen `payment-validation-block.component.ts`
o `session-bill-panel.component.ts`.

## Complexity Tracking

*Sin violaciones de la Constitution Check — sección no aplica.*
