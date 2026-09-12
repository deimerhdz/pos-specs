# Implementation Plan: Pestaña dedicada de Promociones en el menú QR

**Branch**: `081-tab-promociones-menu-qr` | **Date**: 2026-09-12 | **Spec**: [spec.md](./spec.md)

**Input**: Feature specification from `/specs/081-tab-promociones-menu-qr/spec.md`

## Summary

El reconocimiento de código (ver Technical Context) confirma que **toda** la información que
esta funcionalidad necesita ya viaja hoy en `GET /menu` (spec 066): cada `MenuVariantResponse`
trae `promotion: MenuVariantPromotion | null` con `min_qty`, `type`, `value` y los textos ya
compuestos; y el helper `hasPromotion(product)` de `public-menu.component.ts` ya deriva de ahí si
un producto tiene alguna variante en promoción vigente, para la insignia "🎉 Promo". La
funcionalidad es, por eso, **100% frontend, sin ningún cambio en `pos-backend`**:

1. **Pestaña "Promociones" (US1, FR-001 a FR-003, FR-008/FR-009)** — se trata como una categoría
   virtual dentro del mecanismo de pestañas ya existente en `public-menu.component.ts`
   (`activeCategoryId` / `selectCategory` / `visibleProducts`), en vez de introducir un segundo
   sistema de navegación en paralelo. Al seleccionarla, `visibleProducts()` gana una rama que
   recorre todas las categorías y filtra con `hasPromotion()` (research.md D2).

2. **Incremento por cantidad mínima (US2, US3, FR-004 a FR-007, FR-010 a FR-013)** — también sin
   tocar `pos-backend`: `CartItemIn`/`CartItemUpdate.quantity` ya aceptan cualquier entero ≥ 1 y
   el motor de descuentos (`evaluate_variant_sets`, spec 063) ya agrupa en paquetes completos sea
   cual sea la cantidad que reciba, así que basta con que el frontend **elija** qué cantidad
   enviar (research.md D6). Se tocan tres piezas del frontend, todas ya existentes:
   - `product-select.component.ts`: cuando el modal se abre desde "Promociones", la cantidad
     inicial y el paso de `increment()`/`decrement()` pasan de 1 a `min_qty` de la regla vigente
     de la variante elegida (research.md D4); si la variante ya tiene unidades en el carrito no
     múltiplo de `min_qty`, la cantidad a agregar se ajusta para completar el múltiplo más
     cercano hacia arriba (FR-012, research.md D5).
   - `DiningCartService`: gana un mapa cliente, solo en memoria, que recuerda el paso aplicable
     por variante+opciones cuando la línea se creó desde "Promociones" (research.md D3) —
     ningún campo nuevo en el backend ni en `CartResponse`.
   - `cart.component.ts`: los botones +/- de cada línea usan ese paso (o 1 por defecto) en vez de
     sumar/restar siempre 1; bajar exactamente al mínimo retira la línea (FR-006).

Ninguna de las dos piezas introduce entidad, endpoint, migración o dependencia nueva. La única
limitación de alcance explícita ya la fijó `/speckit-clarify`: cuando una regla vigente cubre
varios productos/variantes combinables entre sí (p. ej. "2 entre Mediana y Pequeña"), el paso
por cantidad mínima se aplica **por producto individual dentro de esa tarjeta**, nunca combinando
productos distintos (FR-013) — esa combinación sigue disponible, sin cambios, agregando cada
producto desde su categoría original.

## Technical Context

**Language/Version**: Backend: Python 3.14 / FastAPI 0.136.3 (`pos-backend`) — **sin tocar**.
Frontend: TypeScript / Angular 21 standalone, signals (`pos-heladeria`) — sin cambio de versión.

**Primary Dependencies**: ninguna dependencia nueva en ningún repo (Principio IX: nada que
justificar). Se reutilizan exactamente las mismas piezas ya en producción: backend —
`app/api/v1/menu/schemas.py::MenuVariantPromotion` (spec 066), `app/api/v1/promotions/service.py`
(vigencia y `min_qty`, spec 063), `app/api/v1/cart/schemas.py` (validación de cantidad ya
existente); frontend — `DiningCartService`, `product-select.component.ts`, `cart.component.ts`,
`promotion-pricing.util.ts`.

**Storage**: PostgreSQL 16, schema-per-tenant — **sin migración**. No se agrega ninguna columna
ni tabla; el paso de cantidad por línea (research.md D3) es estado transitorio del navegador, no
persistido.

**Testing**: Backend: `python -m unittest discover -s app/characterization_tests -p 'test_*.py'
-v` — no se toca ningún test con prefijo `"CONGELA comportamiento actual:"` porque no hay cambio
de backend; se confirma en quickstart.md que `test_menu_router.py`/`test_cart_router.py`/
`test_promotions_service.py` siguen en verde sin modificarlos. Frontend: `npm test` (Vitest vía
`@angular/build:unit-test`) — se extienden `public-menu.component.spec.ts`,
`product-select.component.spec.ts` y `dining-cart.service.spec.ts` (ya existen los tres); se crea
`cart.component.spec.ts` (hoy no existe ningún test para ese componente).

**Target Platform**: web, menú QR del comensal (`pos-heladeria`) consumiendo `pos-backend` vía
HTTP/JSON sin cambios de contrato. Sin cambios de infraestructura ni despliegue.

**Project Type**: Web application, dos repositorios independientes en producción — esta spec toca
**únicamente** `pos-heladeria` (frontend). `pos-backend` no gana ni pierde ningún archivo.

**Performance Goals**: sin impacto medible — la pestaña "Promociones" filtra en memoria el mismo
payload que ya carga `GET /menu` una vez por sesión (sin petición adicional); el paso de cantidad
solo cambia qué entero se envía a los mismos endpoints `POST /cart/items` / `PATCH
/cart/items/{id}` que ya existen.

**Constraints**: FR-009/FR-010 (esta misma spec) — la navegación por categoría y el agregado
libre desde ella no cambian; ningún dato histórico se toca (Principio VII, no aplica: no hay
ventas ni facturas involucradas).

**Scale/Scope**: 1 repositorio (`pos-heladeria`). Cero archivos de backend. Frontend: 1 constante
nueva (`PROMOTIONS_TAB_ID`) + 1 rama nueva en `visibleProducts()` + 1 botón de navegación en
`public-menu.component.ts`; `product-select.component.ts` cambia el valor inicial y el paso de
`quantity`; `DiningCartService` gana un mapa interno y dos campos derivados en `CartLine`
(`productVariantId`, `optionKey` — ver data-model.md); `cart.component.ts` cambia el `+1`/`-1`
fijo de sus dos botones por el paso resuelto de la línea. Cero componentes nuevos.

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

Evaluado contra la [Constitución v3.0.0](../../.specify/memory/constitution.md).

| Principio | Estado | Evidencia |
|---|---|---|
| I. Las nuevas funcionalidades nacen de un spec | ✅ | [spec.md](./spec.md), clarificada el 2026-09-12 (4 preguntas resueltas), sirve de contrato completo antes de este plan. |
| II. El comportamiento existente sigue protegido | ✅ | FR-009/FR-010 protegen explícitamente la navegación por categoría y el agregado libre fuera de "Promociones"; no se abre ninguna entrada en `registro-de-anomalias.md` porque esta funcionalidad es puramente aditiva (una vista nueva + una regla de cantidad nueva, acotada a esa vista) — no deroga ningún comportamiento vigente en ningún otro flujo. |
| III. Los characterization tests protegen el comportamiento heredado | ✅ N/A | Sin cambios de backend, ningún test `"CONGELA comportamiento actual:"` se toca; quickstart.md exige correr la suite completa antes/después como red de seguridad. |
| IV. Los nuevos specs pueden introducir nuevo comportamiento | ✅ | El paso de cantidad por `min_qty` (FR-004 a FR-007) es comportamiento nuevo, autorizado explícitamente por la spec y sus clarificaciones. |
| V. Nuevas funcionalidades antes que refactorizaciones oportunistas | ✅ | Se descarta a propósito extraer un componente compartido de "stepper" (research.md D3-D4, alternativas consideradas) — solo dos sitios de llamada existen hoy y ambos se extienden en el lugar; tampoco se toca `menu-lookup.ts` (usado por staff/cocina/cobro, fuera de alcance) pudiendo haberse reutilizado ahí — se prefiere un mapa propio de `DiningCartService` para no ampliar la superficie de un helper compartido por otras pantallas que no necesitan este dato. |
| VI. Evolución incremental | ✅ | Las dos piezas del Summary (pestaña / paso de cantidad) son verificables por separado en dos niveles distintos: la pestaña (US1) funciona con el comportamiento de cantidad libre de hoy sin la segunda pieza; y `product-select.component.ts` en aislamiento (T007) no depende del código de US1 para probarse, porque recibe `fromPromotions`/`existingQtyFor` como inputs directos — aunque el flujo de extremo a extremo de la Historia 2 sí requiere que la pestaña de US1 exista para abrir el modal desde ahí (tasks.md, Dependencies). Cero migración, cero cambio de arquitectura, cero refactor mezclado. |
| VII. Compatibilidad con datos históricos | ✅ N/A | No hay ventas, facturas ni cobros involucrados — solo navegación de catálogo y borrador de carrito. |
| VIII. Evolución del modelo de datos | ✅ N/A | Sin migración, sin campo nuevo en ningún modelo persistido; ver data-model.md para el detalle de las interfaces de frontend que sí cambian (ninguna es una tabla). |
| IX. Dependencias nuevas justificadas | ✅ N/A | Ninguna dependencia nueva en ningún repo. |
| X. Verificación obligatoria | ✅ | [quickstart.md](./quickstart.md) define escenarios ejecutables por cada historia de usuario más la verificación cruzada de que la categoría normal no cambia (US3); se extienden/crean los tests unitarios de frontend listados en Technical Context. |
| XI. Decisiones de negocio frente a técnicas | ✅ | Las decisiones de negocio (alcance del incremento forzado, tipos de regla cubiertos, simetría del decremento, ajuste automático al múltiplo) ya están resueltas en spec.md → Clarifications (sesión 2026-09-12); las decisiones técnicas de este plan (dónde vive el mapa de paso, cómo se resuelve la clave variante+opciones, por qué no tocar `menu-lookup.ts`) están en research.md D1-D7, sin mezclarse con las de negocio. |
| XII. Trazabilidad | ✅ | Necesidad (confusión reportada por el dueño sobre unidades mínimas y dispersión de promociones) → spec 081 (FR-001…FR-013) → Clarifications (1 sesión, 4 preguntas) → este plan → research.md (D1-D7) → data-model.md → contracts/ → quickstart.md → tasks.md (siguiente comando) → tests → verificación. |
| XIII. Todo en español de Colombia | ✅ | El único texto de UI nuevo que introduce esta spec — la etiqueta de la pestaña "Promociones" y el aviso de "no hay promociones activas en este momento" (FR-008) — se redacta en español de Colombia, mismo tono que el resto de textos ya existentes en `pos-heladeria`. |

### Re-evaluación post-diseño (Phase 1)

Repasada tras generar data-model.md, contracts/pestana-promociones-comportamiento.md y
quickstart.md. **Sin violaciones nuevas.**

- **V confirmado con más detalle**: el diseño de Phase 1 (data-model.md) mantiene el paso de
  cantidad como un mapa interno de `DiningCartService`, deliberadamente **sin** persistirlo ni
  exponerlo por fuera del servicio — extenderlo a `CartResponse`/`CartItem` del backend se
  evaluó y se descartó (research.md D6) porque el motor de descuentos ya no lo necesita para
  calcular correctamente.
- **VI confirmado**: el contrato único de Phase 1 mapea a las dos piezas del Summary sin
  mezclarlas — la sección "Elegibilidad de producto" cubre la pieza 1 (US1) y "Paso de cantidad"
  cubre la pieza 2 (US2/US3), cada una con su propio criterio de aceptación en quickstart.md.
- **II confirmado**: el diseño no encontró ningún punto donde la pestaña "Promociones" o el paso
  de cantidad necesiten leer o escribir un campo que hoy usen otras pantallas (Terminal, cocina,
  administración) — todo lo nuevo vive dentro de `pos-heladeria/src/app/modules/tables/`.

Ningún gate queda en rojo.

## Project Structure

### Documentation (this feature)

```text
specs/081-tab-promociones-menu-qr/
├── plan.md                                        # Este archivo
├── research.md                                    # Decisiones D1-D7
├── data-model.md                                  # Interfaces de frontend afectadas (sin tablas nuevas)
├── contracts/
│   └── pestana-promociones-comportamiento.md      # Elegibilidad de producto + paso de cantidad
├── quickstart.md                                  # Validación ejecutable por historia (1-3)
├── checklists/
│   └── requirements.md                            # Ya generada por /speckit-specify
└── tasks.md                                        # Salida de /speckit-tasks (no de este comando)
```

### Source Code (repository root)

```text
../pos-backend/
└── (sin cambios — cero archivos tocados por esta spec)

../pos-heladeria/
├── src/app/modules/tables/
│   ├── pages/
│   │   └── public-menu.component.ts               # + PROMOTIONS_TAB_ID, botón de pestaña,
│   │                                                 rama en visibleProducts(), paso al abrir
│   │                                                 product-select con cantidad existente
│   ├── components/
│   │   ├── product-select.component.ts            # quantity inicial y paso = min_qty vigente
│   │   └── cart.component.ts                      # botones +/- usan el paso resuelto de la línea
│   └── services/
│       └── dining-cart.service.ts                 # mapa interno paso por variante+opciones;
│                                                     CartLine gana productVariantId/optionKey
└── src/app/modules/tables/{pages,components,services}/*.spec.ts   # tests extendidos/nuevos
```

**Structure Decision**: Web application de dos repositorios ya en producción (constitución,
"Alcance del Proyecto"), referenciados como `../pos-backend` y `../pos-heladeria` desde
`pos-specs`. Esta funcionalidad es la primera de las registradas recientemente que **no** toca
`pos-backend` en absoluto — toda la información y validación necesarias ya existen ahí desde las
specs 063/066.

## Complexity Tracking

*Sin violaciones que justificar — la tabla queda vacía a propósito.*
