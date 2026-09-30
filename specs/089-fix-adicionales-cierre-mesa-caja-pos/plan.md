# Implementation Plan: Correcciones de Adicionales del Menú QR, Cierre de Mesa, Impresión de Caja y Total Inmediato en el POS

**Branch**: `089-fix-adicionales-cierre-mesa-caja-pos` | **Date**: 2026-09-30 | **Spec**: [spec.md](./spec.md)

**Input**: Feature specification from `/specs/089-fix-adicionales-cierre-mesa-caja-pos/spec.md`

## Summary

Cinco correcciones, cada una con causa distinta y verificable por separado. Todo lo siguiente sale de leer el código de `pos-backend` y `pos-heladeria` (ambos en `develop`, árbol limpio):

1. **Adicionales del Menú QR (Historias 1 y 3, P1/P2).** Hoy `unit_price` guarda "una unidad" = presentación + adicionales, y **once** sitios lo multiplican por la cantidad del producto ⇒ 2 hamburguesas con 1 tocino cobran 2 tocinos. Se añaden **4 columnas aditivas con default que reproduce el comportamiento actual** (`addons_total` en `cart_items`/`order_items`; `per_line` en `cart_item_options`/`order_item_options`) y una fórmula única `line_total = unit_price × quantity + addons_total`. Las líneas históricas (default `0`/`false`) no cambian ni un centavo; solo `cart/service.py` (Menú QR) escribe la regla nueva. El **consumo de inventario** usa la misma marca por fila para que descuento y reversa cuadren. La edición de adicionales **ya la soporta el backend** (`PATCH /cart/items/{id}`); el trabajo es frontend (reutilizar `ProductSelectComponent` con `initialSelection`).
2. **Cierre de mesa (Historia 4, P2).** `close_table_sessions` es el único que cierra sesiones; el evento `session.closed` ya existe pero **`release_table` y el cierre de sesión vacía no lo emiten**. Se añade un helper de notificación tras el commit en esos dos caminos, una cabecera `X-Session-State: closed` en los dos 401 de cierre (sin tocar cuerpo ni mensajes) y una vista `closed` en el Menú QR con el texto exigido. Reabrir la mesa ya **no** revive la sesión (verificado); se congela con un test. Segunda clarificación (FR-015a/b/c, FR-017a): el cierre ejecuta **el mismo borrado que "Salir"** (`DinerTokenStore.endAccess`, un solo punto) y deja la **marca mínima por pestaña**, de modo que recargar muestre una única pantalla de acceso denegado y nunca abra una sesión nueva; el rechazo del token anterior (401 + `X-Session-State`) ya existe en el backend y solo se congela con pruebas (research D9b).
3. **Impresión del cierre de caja (Historia 5, P3).** No hay PDF de servidor: es `window.print()`. La causa exacta **se reproduce primero** (FR-024). Hipótesis principal: el shell `h-screen overflow-hidden` recorta el contenido al imprimir (más `lg:ml-64` y colores heredados del tema). Corrección acotada por `body.printing-cash-report` para no tocar recibos ni la hoja de QR.
4. **Total inmediato en el POS (Historia 2, P1).** La causa raíz es que `saveOrder()` **no refresca `checkoutPreview`**; el modal "El total cambió" (3 sitios) era el mecanismo de descubrimiento. Se refresca al guardar con una **estimación local** aditiva que el servidor confirma (Cobrar deshabilitado mientras solo haya estimado), se retiran los tres modales y el cambio de última hora pasa a **aviso no bloqueante + segundo Cobrar**. El mensaje de producto agotado nombra el producto (único cambio de backend de esta historia).
5. **Registro y verificación.** A-94, A-95 y A-96 ya están en `registro-de-anomalias.md`; el plan añade **A-97** (hallazgo documental, sin corrección: `set_assignments` pierde `OrderItemOption.quantity` al repartir por unidades).

Sin dependencias nuevas. Una migración de Alembic (metadatos, instantánea). Un ajuste de configuración (`QR_ADDONS_PER_LINE`, por defecto activo) como **bandera de reversa**.

## Technical Context

**Language/Version**: Python 3.12 (`pos-backend`, sin cambio). Angular 20 / TypeScript (`pos-heladeria`, sin cambio).

**Primary Dependencies**: FastAPI, SQLAlchemy 2.0, Pydantic v2, Alembic, Redis (SSE) en backend; Angular signals, Tailwind, RxJS en frontend. **Ninguna dependencia nueva** (Principio IX).

**Storage**: PostgreSQL 16, esquema por tenant. **Cambio de esquema**: `cart_items.addons_total`, `order_items.addons_total` (`NUMERIC(12,2) NOT NULL DEFAULT 0`) y `cart_item_options.per_line`, `order_item_options.per_line` (`BOOLEAN NOT NULL DEFAULT false`), más `CHECK (addons_total >= 0)`. Sin backfill. Ver [data-model.md](./data-model.md).

**Testing**: backend `unittest` (`python -m unittest discover -s app/characterization_tests -p 'test_*.py'`; sesiones SQLite en memoria vía `fixtures.py::new_session`). Frontend `ng test` (Karma/Jasmine) y verificación **manual** de la impresión (ninguna prueba unitaria ejecuta la vista previa real). Guía en [quickstart.md](./quickstart.md).

**Target Platform**: Linux server (contenedor Docker existente) + navegador móvil (Menú QR) y de escritorio/tableta (POS). Impresión: Chrome y Firefox.

**Project Type**: web — `pos-backend` + `pos-heladeria`; la spec vive en `pos-specs`.

**Performance Goals**: total del POS visible en < 0,2 s tras agregar un producto (estimación local); pantalla de gracias < 5 s desde el cierre (SSE); en el barrido con pedidos por cobrar, ≤ 10 s por el sondeo de respaldo con pestaña visible. Sin costo adicional por petición en backend (una columna y una suma por línea).

**Constraints**:
- **Principio VII**: ninguna línea, pedido, venta ni factura anterior cambia de total, comanda o descuento (defaults `0`/`false`, sin `UPDATE`).
- **Deduct == reverse**: el consumo por línea se decide por la marca `per_line` de la fila, nunca por `pricing_type` vigente.
- **Eventos tras el commit**, nunca antes (patrón vigente).
- **Contratos HTTP aditivos**: ningún campo cambia de tipo ni se elimina; los 401 conservan cuerpo y mensajes.
- **Nunca cobrar un importe que el cajero no vio** (FR-028): el cobro exige cifra confirmada y un segundo clic si cambió.

**Scale/Scope**:
- `pos-backend`: `catalog_engine/{core,consumption}.py`, `models/{cart_item,order_item}.py`, migración nueva, `api/v1/cart/{schemas,service,router}.py`, `api/v1/orders/{schemas,checkout,consolidation,kitchen,service,tables_advanced}.py`, `api/v1/table_sessions/service.py`, `api/v1/sales/builder.py`, `core/{qr_context,events,config}.py`, `main.py` (CORS), script `fold_line_addons.py`, tests nuevos y 5 `CONGELA` actualizados (A-94).
- `pos-heladeria`: `dining-cart.service.ts`, `cart.component.ts`, `public-menu.component.ts`, `product-select.component.ts`, `diner.service.ts`, `diner-token.store.ts`, `pos-terminal.store.ts`, `pos-checkout-panel.component.ts`, `payment-attempt-review-panel.component.ts`, `manual-order-page.component.ts`, `pos-order-panel.component.ts`, interfaces, `dashboard-layout.component.ts` + `cash-report.component.ts` (impresión) y sus `.spec.ts`.
- `pos-specs`: A-97 y `implementation-notes.md`.
- 5 historias (la 4 con las clarificaciones 1 y 2): P1×2 (adicionales, total inmediato) · P2×2 (edición, cierre de mesa) · P3 (impresión).

## Constitution Check

*GATE: debe pasar antes de la Fase 0. Re-evaluado tras la Fase 1 (final de la sección).*

| Principio | Evaluación |
|---|---|
| I. Las Nuevas Funcionalidades Nacen de un Spec | ✅ Pass — `spec.md` con clarificaciones (dos sesiones), checklist de calidad y 31 FR. |
| II. El Comportamiento Existente Sigue Protegido | ⚠️ Pass **condicionado** — cambia comportamiento de forma deliberada y pedida por el negocio, ya registrada **antes** de implementar: **A-94** (adicionales una vez por línea en el Menú QR), **A-95** (todo cierre notifica; pantalla de gracias siempre) y **A-96** (se retira el modal "El total cambió"). El plan añade **A-97** (hallazgo documental, sin cambio de comportamiento). Dos valores por defecto que la spec dejó al plan y que **se confirman con el negocio** antes de desplegar: (a) el cambio de total de última hora se resuelve con aviso + segundo Cobrar (A-96 ya lo marca "por confirmar"); (b) un 401 por **vencimiento** (inactividad/duración) conserva la pantalla de nombre, y solo el **cierre** lleva a "gracias" (research D9–D10). Además **A-95 se enmendó** (2026-09-30) por la segunda clarificación: "Salir" también cambia (borrado ampliado, gracias al instante, texto de acceso denegado al recargar), y se actualizan sus pruebas de frontend citando A-95 (research D9b). |
| III. Los Characterization Tests Protegen el Comportamiento Heredado | ⚠️ Pass **condicionado** — se actualizan explícitamente (mismo commit, cita a A-94/A-96 y evidencia de suite verde) los tests que congelan el cobro/consumo por unidad del adicional del Menú QR (`test_catalog_line_pricing`, `test_catalog_consumption_plan`, `test_cart_service`, `test_orders_consolidation`, `test_orders_service`) y los del modal en frontend (`pos-checkout-panel`, `pos-terminal.store`, `payment-attempt-review-panel`, `manual-order-page` `.spec.ts`). El comportamiento por unidad de la terminal POS, mostrador, grupos incluidos y líneas históricas **sigue congelado** y se cubre con tests nuevos que lo prueban (línea histórica + línea nueva en el mismo pedido). Ningún otro `CONGELA` cambia. |
| IV. Los Nuevos Specs Pueden Introducir Nuevo Comportamiento | ✅ Pass — comportamiento nuevo definido y acotado; éxito = SC-001…SC-010. |
| V. Nuevas Funcionalidades Antes que Refactorizaciones Oportunistas | ✅ Pass — el helper `chosen_from_rows` reemplaza dos copias de `_item_options` **porque** es lo que propaga `per_line` (no es refactor gratuito). No se corrige el defecto previo de `set_assignments` (A-97, solo se documenta). No se toca el motor de promociones ni la impresión de recibos. |
| VI. Evolución Incremental | ✅ Pass — 5 historias, cada una con su clase de cambio: (1) migración de esquema como **unidad propia y primera**; (2) lógica de dinero/consumo; (3) frontend de edición; (4) eventos/sesión; (5) CSS de impresión; (6) estado del POS. Se entregan y verifican en ese orden; ninguna mezcla migración con cambio de comportamiento en el mismo commit. |
| VII. Compatibilidad con Datos Históricos | ✅ Pass — defaults `0`/`false`; sin `UPDATE`; `sale_items`/facturas intactas; la fórmula es la misma para líneas históricas. Test explícito de que un pedido anterior devuelve el mismo total, comanda y descuento (SC-003). |
| VIII. Evolución del Modelo de Datos | ✅ Pass (con artefacto) — [data-model.md](./data-model.md) especifica columnas, defaults, compatibilidad, migración, **tres niveles de reversión** (bandera `QR_ADDONS_PER_LINE`, retirar código sin borrar columnas, script `fold_line_addons`) y las condiciones en que el `downgrade()` aborta. |
| IX. Dependencias Nuevas Permitidas con Justificación | ✅ Pass (no aplica) — cero dependencias. La bandera es un campo más de `Settings`. |
| X. Verificación Obligatoria | ✅ Pass (planificado) — [research.md](./research.md) D19 fija la matriz; [quickstart.md](./quickstart.md) cubre los 10 criterios de éxito; la impresión se verifica en la vista previa real, no solo leyendo código (FR-024). |
| XI. Decisiones de Negocio Frente a Decisiones Técnicas | ✅ Pass — las decisiones de negocio están en la spec y en A-94/95/96; el plan resuelve el "cómo". Los dos valores por defecto del plan se marcan arriba (fila II). |
| XII. Trazabilidad | ✅ Pass — Necesidad (problemas detectados 2026-09-30) → spec + clarificaciones → A-94/95/96 → este plan → research/data-model/contracts/quickstart → `tasks.md` → tests. |
| XIII. Todo en Español de Colombia | ✅ Pass — artefactos y textos de usuario ("¡Gracias por tu visita! Esperamos verte pronto.", avisos de total) en español de Colombia. |
| XIV. Estrategia y Convención de Ramas | ✅ Pass (planificado) — antes de tocar código se crea, desde `develop`, `feat/089-qr-addons-per-line` (esquema, dinero, consumo y edición); `fix/089-table-close-pos-total-print` (cierre y POS) se crea **desde esa rama** cuando US1 esté lista (comparten archivos y dependencias); y `fix/089-cash-report-print` (impresión) sale de `develop` por no compartir archivos. Todas **en `pos-backend` y en `pos-heladeria`** cuando aplique. `/speckit-tasks` lo deja como tarea inicial. |
| XV. Política de Commits | ✅ Pass (planificado) — commits pequeños por historia y por unidad (migración; modelo/fórmula; consumo; cada superficie de cobro; evento; vista; CSS; store; …), en inglés, Conventional Commits, sin marcas de IA, y solo cuando el usuario lo pida. |

**Complexity Tracking**: sin violaciones de diseño que justificar. Las filas II y III son prerrequisitos de proceso (registro previo; actualizar tests citando la decisión).

**Re-chequeo post Fase 1**: el diseño no añadió dependencias ni variables de entorno obligatorias, y mantuvo la migración como metadatos. Dos hallazgos quedaron **absorbidos** sin cambiar el veredicto: (1) `OrderItemResponse` **no** exponía `line_total` y el frontend lo calculaba con `unit_price × quantity` en tres sitios ⇒ se añade `line_total` y `addons_total` (aditivos) y el frontend migra sus lecturas; (2) el barrido del scheduler con pedidos por cobrar **no cierra la sesión** (solo a los comensales), así que el aviso llega por el 401 del sondeo, cubierto por la cabecera `X-Session-State`. Orden de despliegue: **backend con migración primero**, luego frontend (el frontend nuevo tolera respuestas sin los campos nuevos usando `line_total ?? unit_price × quantity + addons_total`).

## Project Structure

### Documentation (this feature)

```text
specs/089-fix-adicionales-cierre-mesa-caja-pos/
├── plan.md                        # Este archivo (/speckit-plan)
├── research.md                    # Fase 0 (/speckit-plan)
├── data-model.md                  # Fase 1 (/speckit-plan)
├── quickstart.md                  # Fase 1 (/speckit-plan)
├── contracts/                     # Fase 1 (/speckit-plan)
│   ├── addons-per-line.md         #   Fórmula, consumo, campos de respuesta, edición, sitios a migrar
│   ├── session-closed.md          #   Evento session.closed (matriz), X-Session-State, vista "closed"
│   └── pos-total-immediate.md     #   Estados estimado/confirmado, Cobrar sin modal, agotado con nombre
├── checklists/
│   └── requirements.md            # (ya existente)
└── tasks.md                       # Fase 2 (/speckit-tasks — NO lo crea /speckit-plan)
```

### Source Code (repositorios `pos-backend` y `pos-heladeria`)

```text
pos-backend/
├── alembic/versions/
│   └── <rev>_089_line_addons.py              # NUEVO — 4 columnas + CHECK; down_revision = deb68185a606; @for_each_tenant_schema
├── app/
│   ├── core/
│   │   ├── config.py                         # + QR_ADDONS_PER_LINE: bool = True
│   │   ├── events.py                         # docstring de session_closed: reason ∈ paid|swept|released|empty
│   │   ├── qr_context.py                     # 401 de cierre con header X-Session-State: closed; _abandon_expired notifica
│   │   └── (main.py)                         # CORS expose_headers += "X-Session-State"   (qr_context: sin más cambios para FR-017a; solo tests)
│   ├── models/
│   │   ├── cart_item.py                      # CartItem.addons_total, CartItemOption.per_line
│   │   └── order_item.py                     # OrderItem.addons_total (+ line_total property), OrderItemOption.per_line
│   ├── catalog_engine/
│   │   ├── core.py                           # ChosenOption.per_line=False; compute_unit_price / compute_addons_total / line_total;
│   │   │                                     #   compute_line_price SIN cambio de firma
│   │   └── consumption.py                    # plan_line_consumption: per_line ⇒ per_unit × chosen.quantity (sin × qty)
│   ├── api/v1/
│   │   ├── cart/
│   │   │   ├── schemas.py                    # CartItemResponse.addons_total; CartItemOptionResponse.per_line
│   │   │   ├── service.py                    # add_item/update_item: marcar per_line (grupos con_recargo, si la bandera); unit_price base + addons_total;
│   │   │   │                                 #   serialize_cart/_cart_promo_lines/_cart_consumption/submit_cart usan line_total y copian marcas
│   │   │   └── router.py                     # total del evento order.created con line_total
│   │   ├── orders/
│   │   │   ├── schemas.py                    # OrderItemResponse.addons_total + line_total; OrderItemOptionResponse.per_line
│   │   │   ├── checkout.py                   # chosen_from_rows (reemplaza _item_options); compute_bill/order_sale_lines con addons; release_table sin cambio
│   │   │   ├── consolidation.py              # consolidate_table copia addons_total/per_line; add_item_to_order: agotado con nombre
│   │   │   ├── kitchen.py                    # usa chosen_from_rows; reemplazo con agotado con nombre
│   │   │   ├── service.py                    # create_order: agotado con nombre
│   │   │   ├── tables_advanced.py            # vía SaleLine
│   │   │   └── router.py                     # release_table: notify_sessions_closed(reason="released") tras el commit
│   │   ├── table_sessions/service.py         # notify_sessions_closed; try_release_if_empty -> list[TableSession];
│   │   │                                     #   set_assignments: addons solo en la fila original (A-97 anotado, sin corregir el defecto de quantity)
│   │   └── sales/builder.py                  # SaleLine(addons_total=…): line_total, base_unit_price solo con opciones per_line=False; snapshot JSONB per_line
│   ├── scripts/
│   │   └── fold_line_addons.py               # NUEVO — emergencia: pliega addons de cart_items; simulación por defecto, --apply
│   └── characterization_tests/
│       ├── test_catalog_line_pricing.py      # ACTUALIZAR + casos nuevos (33.000 / 48.000 / adicional x2 / promoción / histórica)  — A-94
│       ├── test_catalog_consumption_plan.py  # ACTUALIZAR + adicional descuenta 1; grupo incluido por unidad; deduct==reverse  — A-94
│       ├── test_cart_service.py              # ACTUALIZAR + add/update con per_line, total de carrito, edición de adicionales   — A-94
│       ├── test_orders_consolidation.py      # ACTUALIZAR + copia de marcas; línea histórica + nueva en un pedido              — A-94
│       ├── test_orders_service.py            # ACTUALIZAR + agotado con nombre; SaleLine con addons                             — A-94/A-96
│       ├── test_line_addons_migration.py     # NUEVO — defaults de columnas y total histórico intacto
│       ├── test_table_sessions_router.py     # AMPLIAR — release_table y sesión vacía emiten session.closed; X-Session-State; token viejo sigue 401
│       └── test_fold_line_addons.py          # NUEVO — simulación/aplicación/no exactas

pos-heladeria/
└── src/app/
    ├── core/realtime/                                # sin cambio (session.closed ya está en sse-client)
    ├── modules/tables/
    │   ├── interfaces/{dining,diner}.interface.ts    # addons_total, line_total, per_line
    │   ├── services/
    │   │   ├── diner.service.ts                      # call(): leer X-Session-State ⇒ DinerSessionExpiredError.closed
    │   │   ├── diner-token.store.ts                  # endAccess(tableToken): purga pos.diner.* + cookies y deja SOLO la marca por pestaña (Salir y cierre)
    │   │   ├── dining-cart.service.ts                # CartLine con addonsTotal; total por line_total; updateItem(options, notes)
    │   │   └── pos-terminal.store.ts                 # checkoutPreviewEstimate; saveOrder con estimado→confirmado→revertir; lecturas por line_total
    │   ├── components/
    │   │   ├── cart.component.ts                     # "Editar adicionales" por línea; "c/u" con unit_price
    │   │   ├── product-select.component.ts           # @Input addonsPerLine (pie del selector con la regla nueva); edición reutiliza initialSelection
    │   │   ├── pos-checkout-panel.component.ts       # checkout() sin confirm.ask; aviso no bloqueante; Cobrar deshabilitado con estimado
    │   │   ├── payment-attempt-review-panel.component.ts # reconfirmIfTotalChanged sin modal; sin gate totalChangeAck
    │   │   └── pos-order-panel.component.ts          # total de línea por line_total
    │   └── pages/
    │       ├── public-menu.component.ts              # vista 'closed' (gracias, en el momento) y 'exited' (acceso denegado único, tras recargar); expireSession(motivo) + exit() usan endAccess;
    │       │                                         #   ngOnInit: 401 closed ⇒ exited; itemLineTotal por line_total; editar adicionales
    │       └── manual-order-page.component.ts        # confirm() sin modal
    ├── modules/dashboard/layout/dashboard-layout.component.ts   # @media print acotado a body.printing-cash-report (altura/overflow/margen del shell)
    └── modules/cash-register/components/cash-report.component.ts # negro sobre blanco forzado en impresión; break-inside en tablas
                                                              # (+ los .spec.ts de todo lo anterior)
```

**Structure Decision**: aplicación web con dos repositorios. La regla del adicional vive **solo** en `catalog_engine` (`compute_*`, `line_total`, `plan_line_consumption`), la lectura de marcas en un único helper `chosen_from_rows`, y cada sitio de cobro la invoca — no se crea ningún paquete nuevo. El cierre de sesión conserva `close_table_sessions` como único punto que cambia el estado y añade un único helper de notificación posterior al commit. El estado del total del POS vive en `PosTerminalStore` (ya es su dueño), no en los componentes.

## Orden de ejecución sugerido (para `/speckit-tasks`)

1. **Preparación**: ramas (XIV), registrar A-97, confirmar con el negocio los dos valores por defecto (fila II).
2. **Diagnóstico de impresión** (paralelizable): reproducir la hoja en blanco y anotar la causa.
3. **Historia 1 — backend**: migración → modelos → fórmula/`ChosenOption` → consumo → `cart/service` → sitios de `line_total` → tests (`A-94`).
4. **Historia 1 — frontend**: lecturas por `line_total` en las cinco superficies.
5. **Historia 3**: "Editar adicionales" (frontend) sobre el `PATCH` existente.
6. **Historia 4**: helper de notificación + `release_table` + `try_release_if_empty` + header + vista `closed` + **`endAccess` compartido con "Salir", marca por pestaña y pantalla única de acceso denegado** (D9b) + pruebas de token anterior con mesa libre/cerrada/reocupada.
7. **Historia 2**: agotado con nombre (backend) + estimado/confirmado + retirar tres modales (`A-96`).
8. **Historia 5**: CSS de impresión según la causa confirmada + verificación en vista previa.
9. **Cierre**: suites completas de ambos repositorios, quickstart completo, `implementation-notes.md`.
