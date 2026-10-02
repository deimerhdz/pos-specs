---

description: "Tareas de la spec 092 — resumen del pedido y total destacado en la pantalla de datos de pago del comensal"
---

# Tasks: Resumen del Pedido en la Pantalla de Pago por Transferencia

**Input**: Documentos de diseño de `/specs/092-resumen-pedido-transferencia/`

**Prerequisites**: [plan.md](./plan.md), [spec.md](./spec.md), [research.md](./research.md) (D1–D11), [data-model.md](./data-model.md), [contracts/checkout-order-summary.md](./contracts/checkout-order-summary.md), [quickstart.md](./quickstart.md)

**Tests**: **sí se piden** (Principio X; plan.md → *Testing*; research.md D10). Frontend únicamente: Vitest 4.0 + `TestBed` (jsdom) vía `npm test` en `pos-heladeria`. Cada prueba se escribe **antes** de su implementación y debe fallar primero. Lo que jsdom no puede medir —desplazamiento real (SC-003), ancho de 320 px (SC-009) y anuncio de lector de pantalla (SC-006)— se verifica a mano y se registra; **no** se simula con un test verde que no prueba nada (research.md D10).

**Organización**: por historia de usuario de [spec.md](./spec.md), en el orden de prioridad del plan: US1 (P1) → US2 (P2) → US3 (P2) → US4 (P3). Cada historia es un incremento entregable por separado; US1 entrega valor sola.

## Formato: `[ID] [P?] [Story] Descripción`

- **[P]**: se puede ejecutar en paralelo (archivo distinto, sin dependencia de una tarea sin terminar)
- **[Story]**: historia de usuario (US1–US4)
- Abreviaturas de ruta: `FE/` = `../pos-heladeria/src/app/` · `CHK/` = `FE/modules/tables/pages/checkout/` · `SPEC/` = esta carpeta (`pos-specs/specs/092-resumen-pedido-transferencia/`)

## Notas de gobernanza

- **Un solo repositorio cambia**: `pos-heladeria`. **`pos-backend` no se toca** — cero endpoints, cero contrato, cero migraciones (plan.md → *Technical Context*). Sus characterization tests de `app/characterization_tests/test_cart_service.py` (que fijan `discounted_total < total` con promoción y `None` sin ella) son el contrato del que sale el "Ahorro" y quedan **intactos** (Principio III).
- **Ramas (Principio XIV)**: `feat/092-payment-step-order-summary` en `pos-heladeria`, creada **desde `develop`** (hoy está en `develop`, árbol limpio) y **antes** de tocar código. `pos-backend` no lleva rama. La spec vive en `pos-specs`, rama `main`.
- **Decisión de negocio pendiente (research.md D6, Principios II/XI)**: mostrar la fila "Ahorro" en el **paso 1**, que hoy no la tiene, es un cambio visible de comportamiento existente. Se plantea en T004 (temprano, no bloquea US1–US3) y se resuelve en T028, **antes** de implementar US4. La válvula de escape ya está diseñada: el input `showSavings` en `false`.
- **Cambio visible ya autorizado por la spec**: el `<h1>` con el nombre del método baja por debajo del bloque de total (FR-001, research.md D9). Lo exige la propia spec, así que **no** requiere entrada de anomalía — solo queda registrado en `SPEC/implementation-notes.md`.
- **Desvío menor respecto a `plan.md` → *Project Structure***: se añade un sexto archivo de producción, `CHK/checkout-product-count.ts`, con la composición de la cadena del conteo. Razón: FR-006 exige que el bloque de total y el `<summary>` usen **una sola** composición, y US1 debe poder entregarse antes de que exista el componente compartido (Principio VI). Cumple la intención de la invariante 2 del contrato; T010 sincroniza contrato y plan para que el desvío quede trazado (Principio XII).
- **Convenciones de código del repo que se siguen**: `@Input()` con decorador (el proyecto no usa inputs de señal en ninguno de sus 34 componentes con entradas), `standalone: true`, `ChangeDetectionStrategy.OnPush`, comentarios y nombres de test en español de Colombia (Principio XIII), identificadores en inglés salvo la constante que el contrato fija en español (`UMBRAL_EXPANDIDO_POR_DEFECTO`).
- **Commits (Principio XV)**: solo cuando el usuario lo pida; pequeños, en inglés, Conventional Commits, **sin ninguna marca de autoría de IA**. Agrupación sugerida en T039.
- **Fuera de alcance, por decisión y no por olvido**: migrar `FE/modules/tables/components/cart.component.ts` al componente compartido (research.md D7), modelar impuestos o subtotal (FR-013), recarga en vivo del pedido en el paso 3, y edición del pedido desde esa pantalla.

---

## Phase 1: Setup

**Propósito**: rama, archivo de notas, datos de prueba y planteamiento temprano de la decisión de negocio.

- [X] T001 Crear en `../pos-heladeria` la rama `feat/092-payment-step-order-summary` **desde `develop`** y confirmar con `git status` que el árbol queda limpio; **no** crear rama en `../pos-backend` (ningún archivo suyo cambia)
- [X] T002 [P] Crear `SPEC/implementation-notes.md` (archivo NUEVO, en español de Colombia) con las secciones "Decisiones", "Medición de layout (SC-003)", "Verificación automática", "Verificación manual", "Accesibilidad (SC-006)" y "Limitaciones"; anotar en "Decisiones" que el `<h1>` del método baja por debajo del bloque de total por exigencia de FR-001 (research.md D9) y que D6 queda pendiente de confirmación
- [X] T003 [P] Preparar los prerrequisitos de verificación de `SPEC/quickstart.md` §0 contra la base de datos de desarrollo y anotar el resultado en `SPEC/implementation-notes.md`: **dos** métodos de pago **no-efectivo** activos y con **nombres distintos** — uno solo con datos de texto y otro con un campo `format: 'image'` (QR) —, una mesa con QR activo y **una promoción vigente** que cubra alguna variante del menú (sin ella los escenarios de "Ahorro" no son verificables); marcar todo lo que se cree con el prefijo `[092-test]`. El segundo método **no es opcional**: sin él, FR-020 (el comportamiento aplica a *cualquier* método que exija comprobante, no solo al llamado "Transferencia Bancaria") no es verificable, y el método con QR es además el caso más exigente de la medición de T022
- [X] T004 [P] Plantear al dueño de la spec la decisión de negocio **D6** de `SPEC/research.md` (¿acepta que el paso 1 muestre la fila "Ahorro" cuando hay promoción vigente, cosa que hoy no hace?) y anotar la pregunta y su fecha en `SPEC/implementation-notes.md` → "Decisiones". **No bloquea** US1, US2 ni US3; la respuesta se necesita antes de la Fase 6

**Checkpoint**: rama creada, notas abiertas, datos de prueba listos y D6 planteada. Recién entonces se escribe código.

---

## Phase 2: Foundational (prerrequisitos que bloquean todas las historias)

**Propósito**: los dos miembros nuevos del servicio, la composición única del conteo y el doble de prueba que hoy haría caer el paso 3 en cuanto su plantilla lea el carrito. **Bloquea** US1 (total y conteo), US2 (líneas y Ahorro) y US4 (consumo desde el paso 1).

- [X] T005 [P] Escribir en `FE/modules/tables/services/dining-cart.service.spec.ts` (AMPLIAR el existente, 327 líneas) los casos de `grossTotal` y `savings` siguiendo el helper `load(body)` que ya tiene el archivo: (a) `total: '10000'` **sin** `discounted_total` → `grossTotal() === 10000` y `savings() === 0`; (b) `discounted_total: null` → igual que (a); (c) `total: '10000'`, `discounted_total: '8500'` → `total() === 8500`, `grossTotal() === 10000`, `savings() === 1500`; (d) `discounted_total` **mayor o igual** que `total` → `savings() === 0`, nunca negativo; (e) tras `clear()` → `grossTotal() === 0` y `savings() === 0` — debe fallar (los miembros no existen)
- [X] T006 Añadir a `FE/modules/tables/services/dining-cart.service.ts` los dos miembros **aditivos** de research.md D5: `readonly grossTotal = signal(0)` escrito en `apply()` con `Number(cart.total)` (el valor **sin** descuento, que hoy `apply()` descarta en la línea 232) y `readonly savings = computed(() => Math.max(0, this.grossTotal() - this.total()))`; reiniciar `grossTotal` en `clear()` junto a `total`; **no** cambiar la semántica de `total()` (sigue siendo `effectivePrice(cart.total, cart.discounted_total)`, consumido por tres pantallas fuera de alcance — Principio V); hace pasar T005
- [X] T007 [P] Escribir `CHK/checkout-product-count.spec.ts` (archivo NUEVO): `formatProductCount(1) === '1 producto'`, `formatProductCount(5) === '5 productos'`, `formatProductCount(0) === '0 productos'` y un valor grande con el formato del proyecto; nombres de test en español de Colombia — debe fallar (archivo inexistente)
- [X] T008 Crear `CHK/checkout-product-count.ts` (archivo NUEVO): función pura exportada `formatProductCount(count: number): string` que devuelve `'1 producto'` en singular y `'N productos'` en plural, con un comentario que cite FR-003, FR-006 y research.md D11 (condicional inline, sin `@angular/localize` ni `i18nPlural`); hace pasar T007
- [X] T009 [P] Ampliar el doble de `DiningCartService` de `CHK/transfer-details-step.component.spec.ts` (hoy `{ clear: vi.fn(), clearDiner: vi.fn() }` en la línea 59) a `{ clear: vi.fn(), clearDiner: vi.fn(), lines: signal([]), total: signal(0), count: signal(0), grossTotal: signal(0), savings: signal(0), isEmpty: computed(...) }`, con un helper del archivo que permita construir un carrito de N líneas por caso; ejecutar `npm test` y confirmar que las **261 líneas** del archivo siguen en verde **antes** de tocar la plantilla del paso 3 (research.md D10)
- [X] T010 [P] Sincronizar la documentación con el desvío de estructura: en `SPEC/contracts/checkout-order-summary.md` nombrar `formatProductCount` de `CHK/checkout-product-count.ts` como la composición única de la invariante 2 (hoy dice "la misma propiedad de conteo del componente") y añadir el archivo a la lista de `SPEC/plan.md` → *Source Code*, con la razón (FR-006 + independencia de US1, Principio VI)

**Checkpoint**: `savings` y `grossTotal` existen y están probados, la cadena del conteo tiene una sola fuente, y el espec del paso 3 ya soporta un carrito completo. Las historias pueden empezar.

---

## Phase 3: User Story 1 — Ver cuánto debo transferir, sin buscarlo (Priority: P1) 🎯 MVP

**Goal**: al llegar a la pantalla de datos de pago, el comensal ve el total a pagar de inmediato, como el elemento de mayor jerarquía visual, acompañado del conteo de productos en unidades.

**Independent Test**: armar un pedido, elegir un método de pago que exija comprobante y comprobar que el total aparece sin desplazarse ni expandir nada, por delante del nombre del método, y que coincide hasta el último dígito con el total del paso 1 (`SPEC/quickstart.md` §2).

### Tests for User Story 1

- [X] T011 [US1] Escribir en `CHK/transfer-details-step.component.spec.ts` (AMPLIAR, sobre el doble de T009) los casos del bloque de total: con `total: 23500` y `count: 5` el render contiene el total formateado con `MoneyPipe` y la cadena **"5 productos"**; con `count: 1` dice **"1 producto"**; el nodo del total aparece en el DOM **antes** del `<h1>` con el nombre del método (FR-001, comparar posición con `compareDocumentPosition` o el orden de `querySelectorAll`); el total **no** está dentro de ningún `<details>` (FR-002); el bloque se pinta también cuando `method()` es `null` (el total no depende de que haya cargado el método) — debe fallar

### Implementation for User Story 1

- [X] T012 [US1] En `CHK/transfer-details-step.component.ts`: hacer público el servicio inyectado (`readonly cart = inject(DiningCartService)`, hoy `private` en la línea 172, siguiendo la convención de `review-step.component.ts`), añadir `MoneyPipe` (`FE/shared/money.pipe.ts`) a `imports`, exponer `readonly productCountLabel = computed(() => formatProductCount(this.cart.count()))` importando `formatProductCount` de `./checkout-product-count`, e insertar el bloque de total **inmediatamente después** del `@if (error())` y **antes** del `@if (!method())` (así el total no depende de la carga del método, FR-002): tarjeta con la etiqueta "Total a pagar", la cifra en `text-3xl font-bold` (el `<h1>` del método es `text-lg`, así que la cifra queda siendo el texto más grande de la vista, FR-001) y `{{ productCountLabel() }}` debajo; hace pasar T011
- [X] T013 [US1] En `CHK/transfer-details-step.component.ts`, reordenar la rama `@else` para que el `<h1>{{ method()!.name }}</h1>` y su frase de apoyo queden **debajo** del bloque de total y **encima** del bloque índigo de datos bancarios (FR-001, FR-016, research.md D9), sin tocar el contenido del bloque índigo, la zona de comprobante ni el botón "Enviar pedido" (FR-018)

### Verification for User Story 1

- [X] T014 [US1] Recorrido manual de `SPEC/quickstart.md` §2 (pasos 1–7) en el navegador con un método no-efectivo, anotando en `SPEC/implementation-notes.md` → "Verificación manual" el total del paso 1 y el del paso 3 para evidenciar que coinciden hasta el último dígito (SC-004), y confirmando que el total es el ya descontado cuando hay promoción vigente (FR-004)

**Checkpoint**: US1 entrega valor por sí sola — el comensal ya sabe cuánto transferir. La pantalla todavía no lista los ítems.

---

## Phase 4: User Story 2 — Revisar mi pedido sin perder de vista los datos bancarios (Priority: P2)

**Goal**: una sección "Resumen del pedido" colapsable en la misma pantalla lista todas las líneas con cantidad, producto, presentación, adicionales, nota y valor de línea, más el Ahorro cuando aplique.

**Independent Test**: con un pedido con adicionales y notas, expandir "Resumen del pedido" en la pantalla de datos de pago y ver cada línea completa, sin navegar a ninguna otra pantalla (`SPEC/quickstart.md` §3).

### Tests for User Story 2

- [X] T015 [P] [US2] Escribir `CHK/checkout-order-summary.component.spec.ts` (archivo NUEVO) con **todos** los casos de `SPEC/quickstart.md` §1, importando `UMBRAL_EXPANDIDO_POR_DEFECTO` en vez del literal `3` (research.md D8): 3 líneas con `collapsible: true` → el `<details>` llega con `open`; 4 líneas → llega **sin** `open`; `collapsible: false` → **no** existe ningún `<details>` ni `<summary>` en el render (FR-022); línea con adicionales y nota → aparecen cantidad, producto, presentación, adicionales, nota y total de línea (FR-007); línea sin adicionales y sin nota → esos dos renglones **no** se pintan; `savings: 1500` → aparece la fila "Ahorro" con el monto formateado (FR-012); `savings: 0` → **no** aparece ninguna fila "Ahorro" (FR-012); `savings: 1500` con `showSavings: false` → tampoco aparece (válvula de D6); cualquier combinación → el render **no** contiene "Impuesto", "IVA" ni "Subtotal" (FR-013, SC-007); `count: 5` con 2 líneas → el `<summary>` dice "5 productos" (FR-006, y el umbral se mide en líneas, no en unidades); `count: 1` → "1 producto"; fijando `details.open = true` a mano → el render crece y **ningún** servicio es invocado, porque el componente no tiene salidas (FR-010; en jsdom se fija `open` directamente, el toggle por click del `<summary>` no es confiable — research.md D1) — debe fallar (el componente no existe)
- [X] T016 [P] [US2] Escribir en `CHK/transfer-details-step.component.spec.ts` (AMPLIAR) los casos de integración del paso 3: el render contiene un `app-checkout-order-summary` con `collapsible` en `true`; su nodo está **después** del bloque índigo de datos bancarios y **antes** de la zona de comprobante (FR-016); con 4 líneas el `<details>` del resumen llega sin `open` y con 3 llega con `open`; el bloque de total sigue fuera de ese `<details>` (FR-002); y **la cadena del conteo del bloque de total es idéntica, carácter por carácter, a la del `<summary>` del resumen** en un mismo render, con `count: 5` y con `count: 1` (FR-006, RN-004: es el único test que verifica en una sola pantalla lo que FR-006 prohíbe — dos conteos distintos del mismo pedido; que los dos salgan de `formatProductCount` lo hace probable, no verificado) — debe fallar

### Implementation for User Story 2

- [X] T017 [P] [US2] Añadir el caso `@case ('chevron-down')` a `FE/shared/icon/icon.component.ts` con el trazo `<path d="m6 9 6 6 6-6" />`, junto a los demás casos y **sin modificar ninguno** de los existentes (research.md D3; el set es propio del proyecto, no hay librería externa — cero dependencias nuevas, Principio IX)
- [X] T018 [US2] Crear `CHK/checkout-order-summary.component.ts` (archivo NUEVO) conforme a `SPEC/contracts/checkout-order-summary.md`: `standalone`, `ChangeDetectionStrategy.OnPush`, selector `app-checkout-order-summary`, **sin** `@Output` y **sin** inyectar `DiningCartService` ni `Router`; `@Input({ required: true })` para `lines: CartLine[]`, `total: number` y `count: number`, `@Input()` con default para `savings = 0`, `collapsible = false` y `showSavings = true`; `export const UMBRAL_EXPANDIDO_POR_DEFECTO = 3`; `countLabel` compuesto con `formatProductCount` de `./checkout-product-count` y leído por el `<summary>` (FR-006); estado inicial **congelado en `ngOnInit`** en un campo plano (`initiallyOpen = this.lines.length <= UMBRAL_EXPANDIDO_POR_DEFECTO`) y enlazado con `[attr.open]="initiallyOpen ? '' : null"`, nunca a una expresión viva (research.md D2 — una expresión viva cerraría de golpe el panel que el comensal acaba de abrir, FR-009); **un solo markup** de la lista y los totales en un `<ng-template>` reutilizado desde las dos ramas con `[ngTemplateOutlet]` (research.md D4); `<details class="group ...">` con `<summary class="... flex cursor-pointer list-none [&::-webkit-details-marker]:hidden">` que lleva el título "Resumen del pedido", el `countLabel` y el `app-icon name="chevron-down"` envuelto en un `<span class="transition-transform group-open:rotate-180">`; fila "Ahorro" si y solo si `showSavings && savings > 0`; cifras pintadas tal como llegan con `MoneyPipe`, sin sumar, restar ni redondear (FR-014, RN-001); `truncate` + `min-w-0` en nombres, adicionales y notas (FR-019, SC-009); se conservan los colores y la tipografía del resumen actual del paso 1 (tarjeta blanca redondeada, borde gris, cifras en gris oscuro, secundarios en gris claro); hace pasar T015
- [X] T019 [US2] En `CHK/transfer-details-step.component.ts`, importar `CheckoutOrderSummaryComponent` y ubicarlo **entre** el cierre del bloque índigo de datos bancarios y el `<div class="mt-4">` de la zona de comprobante (FR-016), pasándole `[lines]="cart.lines()"`, `[total]="cart.total()"`, `[count]="cart.count()"`, `[savings]="cart.savings()"` y `[collapsible]="true"`; **no** tocar la zona de comprobante ni el botón de enviar (FR-018); hace pasar T016

### Verification for User Story 2

- [X] T020 [US2] Recorrido manual de `SPEC/quickstart.md` §3 (pasos 1–8), incluyendo las dos fronteras del umbral (3 líneas expandida, 4 contraída), el caso con y sin promoción vigente, la ausencia de filas de impuestos y subtotal, y la línea con varias unidades cuyo adicional se cobra una sola vez (FR-015, spec 089); anotar el resultado en `SPEC/implementation-notes.md` → "Verificación manual"
- [ ] T021 [US2] Verificación de accesibilidad de `SPEC/quickstart.md` §5 pasos 1, 2, 4 y 5 **con un lector de pantalla real**, sin mouse: foco visible en el título con `Tab`, expandir y contraer con `Enter` y con `Espacio`, y anuncio del nombre y del **estado** (contraído/expandido) al navegar y al activar (FR-011, SC-006). **Empezar por Safari + VoiceOver en iPhone**, que es la plataforma objetivo real del comensal y el único entorno donde `display:flex` sobre el `<summary>` puede costar la semántica nativa de *disclosure* en la que SC-006 se apoya por completo (research.md D1); después TalkBack, y NVDA u Orca solo como contraste de escritorio. Registrar en `SPEC/implementation-notes.md` → "Accesibilidad (SC-006)" qué lector y qué versión se usó y **la frase literal que anunció** en cada paso — no "funciona". Si el estado no se anuncia en algún lector, ejecutar T021-b
- [ ] T021-b [US2] **Solo si T021 muestra que el estado no se anuncia** en algún lector: aplicar en `CHK/checkout-order-summary.component.ts` las palancas en **este orden**, volviendo a verificar con el mismo lector tras cada una, y anotando en `SPEC/implementation-notes.md` cuál resolvió — (a) sacar el `flex` del `<summary>` y moverlo a un `<span>` hijo, dejando el `<summary>` en su `display` nativo con `list-none`: en WebKit es el cambio de `display` del propio `<summary>`, no el ocultar el marcador, lo que puede costar el rol; (b) añadir `aria-expanded` sincronizado con el evento nativo `(toggle)` del `<details>` escribiendo **solo** un campo local del componente — esto contradice la prohibición del contrato, así que si se usa hay que corregir el contrato en el mismo commit y dejar escrito por qué; FR-010 sigue a salvo porque el handler no muta el carrito, no pide red y el componente sigue sin `@Output`; (c) último recurso, un texto visible de estado ("Ver" / "Ocultar") en el `<summary>`, que cualquier lector lee sin depender del rol, a costa de ruido visual. Si T021 cumple en todos los lectores, marcar "no aplica"

**Checkpoint**: US1 y US2 funcionan juntas e independientes; el comensal ve cuánto y qué está pagando sin salir de la pantalla.

---

## Phase 5: User Story 3 — Enviar el comprobante sigue siendo lo fácil de esta pantalla (Priority: P2)

**Goal**: el resumen informa sin estorbar — la zona de comprobante y la acción de enviar siguen alcanzables con a lo sumo un desplazamiento, y el flujo de subida no cambia en nada.

**Independent Test**: en un teléfono de 360 × 640 px con un pedido de 6 líneas, medir el desplazamiento hasta la acción de enviar y completar el flujo de adjuntar, reemplazar, quitar y enviar de punta a punta (`SPEC/quickstart.md` §4 y §6).

> **Esta historia no tiene tareas de implementación por defecto.** Su contenido es medición y no-regresión: jsdom no hace layout, así que SC-003 y SC-009 solo se pueden afirmar midiéndolos (research.md D10). T023 solo se ejecuta si la medición de T022 falla.

- [X] T022 [US3] Medición de layout de `SPEC/quickstart.md` §4 pasos 1–3 en DevTools a **360 × 640** con un pedido de **exactamente 6 líneas** y un método **con QR** (el caso más exigente): confirmar que el resumen llega contraído y evaluar en la consola `document.documentElement.scrollHeight / window.innerHeight` sin haber desplazado nada; **esperado: ≤ 2,0**, es decir ≈ 1.280 px de `scrollHeight` (FR-017, SC-003). Registrar **la cifra con un decimal** en `SPEC/implementation-notes.md` → "Medición de layout (SC-003)", no un veredicto — es el único criterio de la spec cuyo cumplimiento no se puede afirmar sin medirlo, y una cifra es lo que hace la medición reproducible
- [X] T023 [US3] **NO APLICA** — T022 midió **1,2** contra un umbral de 2,0, así que no se usó ninguna palanca de RN-002 (ni compactar el bloque de total, ni reducir el QR, ni tocar el umbral de FR-008). Texto original: **Solo si T022 da más de un desplazamiento**: aplicar en `CHK/transfer-details-step.component.ts` las palancas de RN-002 en **este orden** y volver a medir tras cada una — (a) compactar el bloque de total poniendo la cifra y el conteo en la misma línea; (b) reducir el `max-width` del `<img>` del QR (hoy `max-w-[220px]`, línea 96); (c) bajar el umbral de `UMBRAL_EXPANDIDO_POR_DEFECTO`, que es una **decisión de producto** y se eleva al dueño de la spec, no se cambia por cuenta propia (research.md D9, spec → *Assumptions*). Anotar en `SPEC/implementation-notes.md` qué palanca se usó y la medición final. Si T022 cumple, marcar "no aplica"
- [X] T024 [P] [US3] Casos límite de `SPEC/quickstart.md` §4 pasos 4 y 5: pedido de **10+ líneas** con el resumen expandido → la lista crece, el total sigue arriba y el botón sigue alcanzable al final; **llegar a la pantalla con los dos métodos no-efectivo de T003, uno con QR y otro sin él y con otro nombre** → los dos muestran el total destacado y el resumen, en la misma posición y con la misma jerarquía, sin importar cómo el negocio haya nombrado cada método (FR-020, caso límite "Negocio con varios métodos que exigen comprobante"); registrar en `SPEC/implementation-notes.md` **el nombre de cada método recorrido**, que es la evidencia de que FR-020 se verificó y no se infirió
- [X] T025 [P] [US3] Verificación de ancho mínimo de `SPEC/quickstart.md` §4 paso 6: DevTools a **320 px** con el resumen expandido y nombres de producto y notas largas → sin desplazamiento horizontal y sin contenido recortado de forma ilegible (FR-019, SC-009); registrar en `SPEC/implementation-notes.md`
- [X] T026 [US3] Verificación de FR-010 de `SPEC/quickstart.md` §5 paso 3: expandir y contraer el resumen **varias veces seguidas** con la pestaña Network de DevTools abierta → el pedido no cambia, el total no cambia, no aparece ningún spinner y **no se registra ninguna petición nueva** (US3 esc. 4); registrar en `SPEC/implementation-notes.md`
- [X] T027 [US3] No-regresión completa del flujo de comprobante de `SPEC/quickstart.md` §6 (pasos 1–7): adjuntar, reemplazar, quitar (el botón de enviar queda deshabilitado), adjuntar de nuevo y **enviar** hasta el paso de confirmación; rehidratación tras recargar el navegador con un comprobante ya subido, conviviendo con el resumen sin interferirse; copiar campo y descargar QR siguen funcionando; volver y salir sin enviar no crean ningún pedido (FR-018, SC-005); registrar en `SPEC/implementation-notes.md`

**Checkpoint**: la pantalla informa sin haber empeorado lo que el comensal vino a hacer. SC-003, SC-005 y SC-009 quedan medidos, no asumidos.

---

## Phase 6: User Story 4 — El mismo resumen en los dos pasos del checkout (Priority: P3)

**Goal**: el paso de revisión deja de tener su propio markup y consume el mismo componente, de modo que los dos pasos no puedan divergir.

**Independent Test**: comparar lado a lado el resumen del paso 1 y el del paso 3 con el mismo pedido, verificando líneas, orden, formato de montos, Ahorro y Total (`SPEC/quickstart.md` §7).

> ⚠️ **T028 bloquea esta fase.** Es la única tarea de toda la spec que depende de una decisión de negocio sin tomar.

- [X] T028 [US4] Resolver la decisión **D6** planteada en T004, **antes** de tocar `review-step.component.ts`: si el dueño de la spec **acepta** la fila "Ahorro" en el paso 1 → registrar la anomalía **A-103** en `../pos-specs/specs/000-reconocimiento/registro-de-anomalias.md` siguiendo el formato de A-99…A-102 (qué cambia, por qué, quién decidió, cuándo, qué se ve afectado; la numeración A-103 se confirmó libre) y corregir en `SPEC/spec.md` → *Impacto sobre Funcionalidades Existentes* la frase "cambia de forma estructural, no visual… cualquier diferencia visible en ese paso respecto a hoy es una regresión" para que no contradiga a FR-012/FR-021. Si **no** la acepta → el paso 1 pasa `[showSavings]="false"` en T030 y la excepción a US4 esc. 1 se registra explícitamente en `SPEC/spec.md`. En ambos casos, anotar la decisión y su fecha en `SPEC/implementation-notes.md` → "Decisiones" (Principios II y XI)
- [X] T029 [P] [US4] Escribir `CHK/review-step.component.spec.ts` (archivo NUEVO) como **base de no-regresión del paso 1, antes de la extracción**: con un carrito de varias líneas (una con adicionales y nota, otra sin ninguno de los dos) fijar el formato de cada renglón (`2×`, producto `·` presentación, adicionales unidos con `", "`, nota entre comillas), el **orden** de las líneas tal como llegan, la fila "Total" con el valor de `cart.total()` formateado, y que el botón "Elegir método de pago" queda deshabilitado con el carrito vacío; **sin** el prefijo `"CONGELA comportamiento actual:"` en los nombres de test, porque la propia spec autoriza una diferencia en esta pantalla (la fila "Ahorro", research.md D10) y congelar algo que el mismo spec va a cambiar obligaría a editar el test en el commit que lo crea; ejecutar `npm test` y confirmar que pasa **contra el código actual**, sin tocarlo
- [X] T030 [US4] Extraer el resumen de `CHK/review-step.component.ts`: reemplazar la tarjeta `<div class="bg-white rounded-2xl ...">` completa (líneas 33–55: el `@for` de líneas y la fila "Total") por `<app-checkout-order-summary [lines]="cart.lines()" [total]="cart.total()" [count]="cart.count()" [savings]="cart.savings()" />` —**sin** pasar `collapsible`, que por default es `false` (FR-022)— más `[showSavings]="false"` únicamente si T028 resolvió que no; conservar el `<h1>Tu pedido</h1>`, el encabezado fijo y el botón "Elegir método de pago"; añadir `CheckoutOrderSummaryComponent` a `imports` y retirar de ahí `MoneyPipe` si deja de usarse en la plantilla (`IconComponent` sigue en uso por el botón de salir)
- [X] T031 [US4] Ejecutar `npm test` y confirmar que `CHK/review-step.component.spec.ts` (T029) sigue en verde tras la extracción, ajustando **únicamente** lo que la decisión de T028 autorice (la fila "Ahorro"); cualquier otra diferencia es una regresión y se corrige en el componente, no en el test (Principios II y III)
- [X] T032 [US4] Comparación manual lado a lado de `SPEC/quickstart.md` §7 (pasos 1–4): con el mismo pedido, líneas, orden, formato de montos, Ahorro y Total idénticos en el paso 1 y el paso 3 (FR-021, SC-008); en el paso 1 el resumen está **siempre expandido**, sin chevron y sin control de colapso (FR-022); **sin** promoción vigente, el paso 1 se ve exactamente igual que en `develop`; registrar en `SPEC/implementation-notes.md`

**Checkpoint**: las cuatro historias funcionan y los dos pasos del checkout ya no pueden divergir: hay un solo artefacto.

---

## Phase 7: Polish & Cross-Cutting Concerns

**Propósito**: suite completa, no-regresión fuera del checkout y cierre documental.

- [X] T033 Ejecutar `npm test` completo en `../pos-heladeria` y confirmar la suite **entera** en verde (no solo los archivos de esta spec); anotar el conteo final de tests en `SPEC/implementation-notes.md` → "Verificación automática"
- [X] T034 [P] No-regresión fuera del checkout de `SPEC/quickstart.md` §8 (pasos 1–4): el carrito del menú público (`FE/modules/tables/components/cart.component.ts`) sigue pintando sus líneas y su total igual que antes, porque **no se migró** por decisión explícita (research.md D7); en la terminal del cajero el importe al confirmar el pago coincide con el total que vio el comensal (SC-004, FR-023); ninguna pantalla del cajero, mesero ni administrador cambió; el flujo de pago en **efectivo** no pasa por esta pantalla y sigue enviando el pedido desde el paso de método
- [X] T035 [P] Verificar con `grep -rniE 'impuesto|iva|subtotal' ../pos-heladeria/src/app/modules/tables/pages/checkout/` que ninguna plantilla del checkout introdujo una fila de impuestos ni de subtotal (FR-013, RN-003, SC-007); anotar el resultado en `SPEC/implementation-notes.md`
- [X] T036 [P] Confirmar con `git -C ../pos-backend status --porcelain` que `pos-backend` **no tiene ningún cambio** y que sus characterization tests de `app/characterization_tests/test_cart_service.py` quedaron intactos (Principio III); si apareciera algún cambio, revertirlo: está fuera de alcance
- [X] T037 Cerrar `SPEC/implementation-notes.md` con la lista de cierre de `SPEC/quickstart.md` §9: suite en verde, bloques manuales completos, medición de SC-003 con su resultado real y la palanca usada si hubo, accesibilidad verificada con lector real, y la decisión D6 registrada donde corresponda; añadir la sección "Limitaciones" con las dos fuentes de markup que quedan vivas tras la spec (el componente compartido del checkout y el carrito del menú público, research.md D7) como candidato a spec propia de limpieza
- [X] T038 [P] Limpiar o marcar con `[092-test]` en la base de datos de desarrollo los datos creados para la verificación (métodos de pago de prueba, promoción, pedidos y comprobantes), según `SPEC/quickstart.md` §9
- [X] T039 **Solo si el usuario lo pide** (Principio XV): confirmar en commits pequeños, en inglés, Conventional Commits y **sin marcas de autoría de IA**, con esta agrupación sugerida — (1) `feat` servicio: `grossTotal` y `savings` + su spec; (2) `feat` helper del conteo + su spec; (3) `test` ampliación del doble de `DiningCartService` del paso 3; (4) `feat` bloque de total destacado y reorden del paso 3 (US1); (5) `feat` icono `chevron-down`; (6) `feat` componente `checkout-order-summary` + su spec; (7) `feat` resumen colapsable en el paso 3 (US2); (8) `test` base de no-regresión del paso 1; (9) `refactor` el paso 1 consume el componente compartido (US4); (10) `docs` en `pos-specs` (tasks, notas de implementación, A-103 y corrección de `spec.md`)

---

## Dependencies & Execution Order

### Dependencias de fase

- **Fase 1 (Setup)**: sin dependencias — empieza de inmediato. T001 (rama) va **antes** de cualquier cambio de código (Principio XIV).
- **Fase 2 (Foundational)**: depende de T001. **Bloquea las cuatro historias**: US1 necesita `count` y `formatProductCount`, US2 necesita `savings`, US4 necesita todo lo anterior, y los tres pasos que leen el carrito necesitan el doble ampliado de T009.
- **Fase 3 (US1)**: depende de la Fase 2 completa.
- **Fase 4 (US2)**: depende de la Fase 2. Técnicamente podría ir en paralelo con US1, pero **comparte archivo** con ella (`transfer-details-step.component.ts`), así que en un solo desarrollador va después; T017 (icono) y T015 (spec del componente) sí son paralelizables desde el inicio de la fase.
- **Fase 5 (US3)**: depende de que US1 y US2 estén en la pantalla — mide la pantalla final, no una intermedia.
- **Fase 6 (US4)**: depende de la Fase 4 (el componente debe existir) y de **T028** (decisión de negocio).
- **Fase 7 (Polish)**: depende de todas las fases que se decidan entregar.

### Dependencias entre historias

- **US1 (P1)**: independiente. Entrega valor sola: el comensal ya sabe cuánto transferir aunque el resumen nunca se implemente.
- **US2 (P2)**: independiente en lo funcional, pero **se apoya** en US1 para que el acordeón pueda estar contraído sin ocultar lo crítico (el total vive fuera del `<details>`).
- **US3 (P2)**: no añade comportamiento; **verifica** la pantalla que dejan US1 y US2. Sin ellas no hay nada que medir.
- **US4 (P3)**: depende del componente de US2 y de la decisión T028. Es la única historia con riesgo de regresión, y va al final con su red de tests puesta antes (T029).

### Dentro de cada historia

- Los tests se escriben **primero** y deben fallar antes de implementar.
- Servicio y helpers antes de componentes; componente compartido antes de sus consumidores; consumidores antes de la verificación manual.
- La base de no-regresión del paso 1 (T029) va **antes** de la extracción (T030), nunca después.

### Oportunidades de paralelismo

- Fase 1: T002, T003 y T004 en paralelo tras T001.
- Fase 2: T005 ∥ T007 ∥ T009 ∥ T010 (archivos distintos); T006 depende de T005 y T008 de T007.
- Fase 4: T015 ∥ T016 ∥ T017 (tres archivos distintos) antes de T018 y T019.
- Fase 5: T024 ∥ T025 (dos emulaciones distintas); T022 va antes de T023.
- Fase 7: T034 ∥ T035 ∥ T036 ∥ T038.
- Con varios desarrolladores: tras la Fase 2, uno puede tomar el componente compartido (T015+T017+T018) mientras otro hace US1 en el paso 3 (T011–T013); se integran en T019.

---

## Parallel Example: Fase 2 (Foundational)

```bash
# Las cuatro tareas tocan archivos distintos y pueden ir juntas:
Tarea: "T005 tests de grossTotal/savings en dining-cart.service.spec.ts"
Tarea: "T007 tests de formatProductCount en checkout-product-count.spec.ts"
Tarea: "T009 ampliar el doble de DiningCartService en transfer-details-step.component.spec.ts"
Tarea: "T010 sincronizar contrato y plan con el archivo nuevo del conteo"
```

## Parallel Example: Fase 4 (US2)

```bash
# Antes de escribir el componente, tres archivos independientes:
Tarea: "T015 spec del componente checkout-order-summary (debe fallar)"
Tarea: "T016 tests de integración del resumen en el paso 3 (deben fallar)"
Tarea: "T017 icono chevron-down en icon.component.ts"
```

---

## Implementation Strategy

### MVP primero (solo US1)

1. Fase 1 (Setup) — rama y notas.
2. Fase 2 (Foundational) — **crítica, bloquea todo**.
3. Fase 3 (US1) — bloque de total destacado.
4. **PARAR Y VALIDAR**: recorrido de `quickstart.md` §2. El comensal ya puede transferir el monto correcto.
5. Desplegable aquí: resuelve el riesgo concreto de la spec (transferir un monto equivocado) sin haber tocado el paso 1 ni el markup de las líneas.

### Entrega incremental

1. Setup + Foundational → base lista.
2. + US1 → validar → desplegar (**MVP**).
3. + US2 → validar → desplegar (el comensal ya confirma qué está pagando).
4. + US3 → medir SC-003/SC-009 y el flujo de comprobante → desplegar con la medición registrada.
5. + US4 → resolver D6, extraer con red de tests → desplegar.
6. Fase 7 → cierre documental y limpieza de datos de prueba.

Cada incremento suma valor sin romper el anterior. El de mayor riesgo (US4, que toca una pantalla que hoy funciona) es el último y el único con una base de no-regresión escrita a propósito.

### Si hay que recortar alcance

- **US4 es la primera candidata a posponer**: su valor es estructural (evitar divergencia futura), no visible para el comensal. Posponerla deja dos markups del resumen en el checkout —exactamente la deuda que FR-021 quiere cerrar— así que se registra como pendiente, no se olvida.
- **US3 no se puede recortar**: no es trabajo de implementación, es la verificación de que US1 y US2 no empeoraron la pantalla (RN-002). Saltarla equivale a afirmar SC-003 sin medirlo.

---

## Notes

- `[P]` = archivos distintos, sin dependencia pendiente.
- `[Story]` mapea cada tarea a su historia para trazabilidad (Principio XII).
- Los tests se verifican en rojo antes de implementar; un test que pasa de entrada no está probando lo que cree.
- Ninguna tarea de esta spec toca `pos-backend`, ninguna añade dependencias y ninguna cambia el contrato de `GET /cart`.
- El componente compartido **no tiene salidas** a propósito: es lo que hace que FR-010 no se pueda violar ni por accidente ni por un cambio futuro descuidado. Añadirle un `@Output` más adelante es un cambio de alcance que necesita su propio spec.
- Evitar: duplicar el markup de líneas dentro del propio componente (research.md D4), enlazar `[attr.open]` a una expresión viva (D2), y escribir un test de Vitest que pretenda verificar desplazamiento o anuncio de lector de pantalla (D10).
