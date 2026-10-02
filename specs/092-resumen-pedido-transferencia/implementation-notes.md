# Notas de implementación — spec 092: Resumen del Pedido en la Pantalla de Pago

**Spec**: [spec.md](./spec.md) · **Plan**: [plan.md](./plan.md) · **Research**: [research.md](./research.md) · **Tareas**: [tasks.md](./tasks.md)

**Rama de código**: `feat/092-payment-step-order-summary` en `pos-heladeria`, creada desde `develop` el 2026-10-02 con el árbol limpio (Principio XIV). **`pos-backend` no se toca**: cero endpoints, cero contrato, cero migraciones.

---

## Decisiones

### D6 — La fila "Ahorro" se muestra en los dos pasos del checkout ✅ RESUELTA

- **Planteada** (T004): 2026-10-02. Pregunta al dueño de la spec: *¿acepta que el paso 1 ("Revisa tu pedido") muestre la fila "Ahorro" cuando hay promoción vigente, cosa que hoy no hace?*
- **Respondida** (T028): 2026-10-02. **Sí — se muestra en ambos pasos.**
- **Consecuencias aplicadas**:
  - El paso 1 **no** pasa `showSavings`; queda en su default `true`.
  - Se registró la anomalía **A-103** en `../pos-specs/specs/000-reconocimiento/registro-de-anomalias.md` (Principio II: cambio visible de comportamiento existente con decisión de negocio detrás).
  - Se corrigió en [spec.md](./spec.md) → *Impacto sobre Funcionalidades Existentes* la frase que decía que el paso 1 "cambia de forma estructural, no visual" y que "cualquier diferencia visible en ese paso respecto a hoy es una regresión", porque contradecía a FR-012 / FR-021 / US4 esc. 1.
- **Alcance real del cambio visible**: solo se manifiesta **con promoción vigente aplicada**. Sin promoción, `savings() === 0` y la fila no se pinta, así que el paso 1 se ve exactamente igual que en `develop` — que es el caso por defecto.
- **Válvula de escape conservada**: el input `showSavings` del componente compartido sigue existiendo con default `true`. Si el negocio revierte la decisión, el cambio cuesta un booleano en una plantilla, sin rediseñar nada.

### El `<h1>` del método de pago baja por debajo del bloque de total

Cambio visible de una pantalla existente (paso 3), **ya autorizado por la propia spec**: FR-001 exige que el total sea "el primer dato que se lee bajo el encabezado, **por delante del nombre del método de pago**", y research.md D9 fija el orden vertical resultante. Por eso **no** requiere entrada en el registro de anomalías — no hay contradicción que resolver ni decisión de negocio que tomar, la spec ya la tomó. Queda registrado aquí por trazabilidad (Principio XII).

Orden vertical final del paso 3 (research.md D9):

1. Encabezado fijo existente (volver · indicador de paso · salir) — sin cambios.
2. **Bloque de total a pagar** (nuevo) — la cifra es el texto más grande de la vista.
3. `<h1>` con el nombre del método + frase de apoyo, y el bloque índigo de datos bancarios y QR — **el `<h1>` baja de posición**.
4. Sección `<details>` **"Resumen del pedido"** (nueva).
5. Zona de comprobante y botón "Enviar pedido" — sin cambios de comportamiento.

### Desvío menor respecto a `plan.md` → *Project Structure*

Se añadió un sexto archivo de producción no previsto en el plan: `checkout-product-count.ts`, con la composición única de la cadena del conteo (`formatProductCount`). Razón: FR-006 exige que el bloque de total y el `<summary>` del resumen usen **una sola** composición, y US1 debe poder entregarse antes de que exista el componente compartido (Principio VI). El contrato y el plan se sincronizaron en T010 para que el desvío quede trazado (Principio XII).

---

## Medición de layout (SC-003) ✅ CUMPLE — 2026-10-02

Condiciones exactas de T022: emulación **360 × 640**, pedido de **exactamente 6 líneas**, método **con QR** (Nequi, imagen cargada), resumen en su **estado inicial** (contraído) y `window.scrollY === 0` — sin haber desplazado nada.

```js
document.documentElement.scrollHeight / window.innerHeight   // umbral: ≤ 2,0
```

| Escenario | `scrollHeight` | `innerHeight` | **Ratio** | Umbral | Resultado |
|-----------|---------------:|--------------:|----------:|-------:|-----------|
| **6 líneas, contraído, con QR** (el caso de SC-003) | 779 px | 640 px | **1,2** | ≤ 2,0 | ✅ cumple con margen |
| 6 líneas, **expandido**, con QR | 1.088 px | 640 px | **1,7** | ≤ 2,0 | ✅ cumple |
| 11 líneas, contraído, sin QR | — | 640 px | **1,0** | ≤ 2,0 | ✅ cabe en un pantallazo |
| 11 líneas, **expandido**, sin QR | — | 640 px | **1,7** | ≤ 2,0 | ✅ cumple |
| 3 líneas, expandido (frontera inferior), con QR | — | 640 px | **1,5** | ≤ 2,0 | ✅ cumple |

**No hizo falta ninguna palanca de RN-002.** El caso que la spec identificó como más exigente (6 líneas + QR de 220 px + bloque de total nuevo + `<details>`) queda en 1,2 — es decir, a **779 px** de alto total contra un presupuesto de 1.280 px. Por eso **T023 no aplica**: no se compactó el bloque de total, no se redujo el `max-width` del QR y **no se tocó el umbral de FR-008**, que era la palanca que habría exigido una decisión de producto.

> La estimación de research.md D9 (≈70-80 px del bloque de total + ≈48 px del `<details>` contraído) resultó conservadora frente a la medición real. El riesgo que D9 pedía medir en vez de estimar **no se materializó**.

---

## Verificación automática

**Suite completa de `pos-heladeria`** — `npm test` (T033):

| Hito | Archivos | Tests | Pasan | Fallan | Saltados |
|------|---------:|------:|------:|-------:|---------:|
| Línea base de `develop` (medida con `git stash`, árbol limpio) | 106 | 1.248 | 1.228 | **19** | 1 |
| Después de la spec 092 | 109 | **1.311** | **1.291** | **19** | 1 |
| **Delta** | **+3** | **+63** | **+63** | **0** | 0 |

> **Los 19 fallos son preexistentes de `develop`, no de esta spec.** Se midieron a propósito antes de empezar, con `git stash -u`, para no atribuirle a la 092 algo que ya venía roto. Viven en seis archivos que **esta spec no toca**: `app.spec.ts`, `core/services/auth.service.spec.ts`, `core/services/menu.service.spec.ts`, `super-admin/services/tenant.service.spec.ts`, `components/pos-checkout-panel.component.spec.ts` y `components/pos-order-panel.component.spec.ts`. El conteo de fallos es **idéntico** antes y después: cero regresiones.

> **Inestabilidad preexistente a vigilar (no introducida por esta spec)**: en una de las corridas, esos mismos seis archivos dieron **23** fallos en vez de 19 (coincidió con un rebuild de `ng serve` en paralelo). Tres corridas consecutivas posteriores dieron 19/19/19. Ningún archivo de la spec 092 apareció nunca en la lista de fallos, en ninguna corrida. La causa probable es el entorno compartido entre specs que el propio repo ya documenta (`dining-cart.service.spec.ts` → *"Ver nota en `diner.service.spec.ts`: los specs comparten entorno"*, por eso hace `TestBed.resetTestingModule()` y `localStorage.clear()` en cada `beforeEach`). **Es deuda de la suite heredada, candidata a spec propia**; conviene no confundirla con una regresión al revisar este trabajo.

Archivos que esta spec creó o amplió — **los 63 tests nuevos pasan**:

| Archivo | Estado | Tests | Qué cubre |
|---------|--------|------:|-----------|
| `services/dining-cart.service.spec.ts` | AMPLIADO | +5 | `grossTotal` y `savings` con `discounted_total` ausente, `null`, menor y mayor o igual que `total`, y tras `clear()` (research.md D5) |
| `checkout/checkout-product-count.spec.ts` | NUEVO | 5 | `formatProductCount`: singular, plural, cero, valor grande y que el 2 ya es plural (FR-003, FR-006, research.md D11) |
| `checkout/checkout-order-summary.component.spec.ts` | NUEVO | 25 | FR-005 a FR-010, FR-012 a FR-015, FR-019, FR-021, FR-022, las dos fronteras del umbral y el contrato de accesibilidad |
| `checkout/transfer-details-step.component.spec.ts` | AMPLIADO | +18 | Bloque de total, orden en el DOM frente al `<h1>`, total fuera del `<details>`, ubicación del resumen (FR-016) y la igualdad carácter por carácter de los dos conteos (FR-006, RN-004) |
| `checkout/review-step.component.spec.ts` | NUEVO | 10 | Base de no-regresión del paso 1, escrita **antes** de la extracción (research.md D10) |

**La extracción del paso 1 no obligó a tocar un solo test.** T031 preveía ajustar "únicamente lo que la decisión de T028 autorice"; no hizo falta ningún ajuste, porque los tests de T029 fijan el caso por defecto (`savings: 0`), donde la fila "Ahorro" no se pinta. El conteo de la suite es el mismo antes y después de T030.

**Lo que la suite deliberadamente NO cubre** (research.md D10): desplazamiento real (SC-003), ancho de 320 px (SC-009) y anuncio de lector de pantalla (SC-006). jsdom no hace layout ni habla con un lector, así que un test verde sobre eso no probaría nada y sería peor que no tenerlo. Se verifican a mano y se registran abajo.

### Grep de FR-013 / RN-003 / SC-007 (T035)

El grep literal de la tarea (`grep -rniE 'impuesto|iva|subtotal'`) **da falsos positivos**: `iva` casa dentro de `ActivatedRoute`, `private` y `derivada`. Con límites de palabra, que es lo que la tarea realmente quiere comprobar:

```
grep -rniE '\b(impuestos?|iva|subtotal)\b' pos-heladeria/src/app/modules/tables/pages/checkout/
```

**Una sola coincidencia en código de producción**, y es el comentario de `checkout-order-summary.component.ts` que documenta la prohibición ("Ninguna fila de impuestos ni de subtotal…"). **Cero coincidencias en texto visible** de cualquier plantilla del checkout. Verificado además en el navegador sobre la pantalla real, con y sin promoción, contraído y expandido: `/\b(impuesto|iva|subtotal)\b/i` sobre `document.body.innerText` → `false` en todos los casos.

### Confirmación de que `pos-backend` no cambió (T036)

```
git -C ../pos-backend status --porcelain
```

**Resultado: sin salida** → cero archivos modificados. Los characterization tests de `app/characterization_tests/test_cart_service.py` (que fijan `discounted_total < total` con promoción y `None` sin ella, y que son el contrato del que sale el "Ahorro") quedaron intactos: su último commit sigue siendo el de la spec 089 (`5518322`), anterior a esta spec (Principio III).

### Build de producción

`npm run build` compila sin errores. Los dos únicos avisos (presupuesto de bundle inicial y el `qrcode` CommonJS) son **preexistentes** y no los introduce esta spec.

---

## Verificación manual

> Todos los bloques de abajo requieren el entorno de desarrollo corriendo (backend en `localhost:8000`, `npm start` en `pos-heladeria`), una mesa con QR activo, **dos** métodos de pago no-efectivo con nombres distintos —uno con campo `format: 'image'` (QR) y otro solo de texto— y **una promoción vigente** sobre alguna variante del menú.

### Prerrequisitos de datos (T003) ✅ LISTOS — 2026-10-02

Entorno de desarrollo verificado en vivo: backend en `localhost:8000` (200), frontend en `localhost:4200` (200), Postgres en el contenedor `pos`. Esquema de datos: `heladeria`.

| Prerrequisito | Estado | Detalle |
|---------------|--------|---------|
| Mesa con QR activo | ✅ ya existía | Mesa 1 "mesa de pruebas 1", `qr_token = ffbe572c-7c2e-441f-9a43-76399077b578`, activa |
| Método no-efectivo **con** QR | ✅ ya existía | **"Nequi"** — `payment_info` con `celular` (texto) + `qr` (campo `format: 'image'`). Es el caso más exigente de la medición de T022 |
| Método no-efectivo **solo de texto**, con **otro nombre** | ⭐ **CREADO** | **"Transferencia Bancolombia [092-test]"** — `payment_info` con `cuenta` y `tipo_cuenta`, **sin** campo de imagen. **No es opcional**: sin él, FR-020 (el comportamiento aplica a *cualquier* método que exija comprobante, no solo al llamado "Transferencia Bancaria") no es verificable |
| Promoción vigente | ⭐ **REACTIVADA** | **"test 25"** (`cbbc11e8-1c7f-4249-8b21-7e411754eff1`), `package_price`: **2 unidades de "Granizado del diablo"** a precio de paquete. En la presentación de $15.000 → paquete de $16.000 en vez de $30.000, es decir un **Ahorro de $14.000** bien visible. Las otras dos presentaciones cubiertas dan $10.000 y $8.000 de ahorro |

> **Nota sobre `days_of_week`**: la convención del backend es **lunes = 0** (`app/characterization_tests/test_menu_router.py:118` → `days_of_week="0"  # solo lunes`), no ISO 1-7.

### Limpieza de datos de prueba (T038) ✅ HECHA — 2026-10-02

| Qué | Acción | Estado |
|-----|--------|--------|
| Promoción `test 25` (`cbbc11e8-…`) | **Revertida a sus valores originales**: `ends_at = '2026-09-30 00:00:00'`, `days_of_week = '1,2,3,4,5,6'`, `start_time = '08:17:00'`, `end_time = '14:17:00'` | ✅ idéntica a como estaba |
| Método `Transferencia Bancolombia [092-test]` | **Desactivado** (`active = false`), no borrado | ⚠️ ver abajo |
| Pedido de verificación + su intento de pago | Se conservan, identificables por el método `[092-test]` y por los comensales `Leo 092-test` / `Ana 092-test` | marcados |

**Por qué el método se desactiva en vez de borrarse**: el pedido que se creó para verificar SC-005 tiene un `order_payment_attempts` que lo referencia. Borrarlo rompería la integridad de ese historial. Desactivarlo logra lo que importa —deja de ofrecérsele al comensal— sin falsear datos ya escritos. Si se quiere eliminar del todo, hay que borrar primero el pedido de prueba y su intento de pago.

**El contenedor `pos-api` está en bucle de reinicio** (91 reinicios, `DATABASE_URL` apuntando a `localhost:5432` desde dentro del contenedor). Es **preexistente y ajeno a esta spec**: el backend que responde en `:8000` es un proceso del host (`fastapi dev` con el venv `pos-backend/env`). No se tocó.

**Recorrido ejecutado el 2026-10-02** sobre el entorno de desarrollo real (backend `fastapi dev` en `localhost:8000`, `ng serve` en `heladeria.localhost:4200`, Postgres del contenedor `pos`), conduciendo Chrome con DevTools. Mesa 1 "mesa de pruebas 1". Dos comensales: "Leo 092-test" (pedido con promoción) y "Ana 092-test" (pedido sin promoción).

| Bloque | Tarea | Qué verifica | Estado |
|--------|-------|--------------|--------|
| `quickstart.md` §2 | T014 | US1 — total destacado, conteo en unidades, coincidencia con el paso 1 (SC-004), total ya descontado (FR-004) | ✅ **OK** |
| `quickstart.md` §3 | T020 | US2 — fronteras del umbral, Ahorro con y sin promoción, sin impuestos ni subtotal, adicional cobrado una vez (FR-015) | ✅ **OK** |
| `quickstart.md` §4 p. 1-3 | T022 | SC-003 — medición de desplazamiento (ver tabla arriba) | ✅ **OK (1,2)** |
| `quickstart.md` §4 p. 4-5 | T024 | 10+ líneas expandido y **los dos métodos no-efectivo** (FR-020) | ✅ **OK** |
| `quickstart.md` §4 p. 6 | T025 | SC-009 — 320 px, sin desplazamiento horizontal | ✅ **OK** |
| `quickstart.md` §5 p. 3 | T026 | FR-010 — expandir/contraer varias veces: cero peticiones | ✅ **OK** |
| `quickstart.md` §5 p. 1-2, 4-5 | T021 | SC-006 — teclado y **lector de pantalla real** | ⚠️ **PENDIENTE** (ver "Accesibilidad") |
| `quickstart.md` §6 | T027 | SC-005 — adjuntar, reemplazar, quitar, readjuntar, enviar; rehidratación (FR-018) | ✅ **OK** |
| `quickstart.md` §7 | T032 | US4 — paso 1 vs. paso 3; paso 1 siempre expandido sin chevron (FR-022); sin promoción, paso 1 idéntico a `develop` | ✅ **OK** |
| `quickstart.md` §8 | T034 | FR-023 — carrito del menú público intacto (D7), importe del cajero coincide (SC-004) | ✅ **OK** |

### Evidencia por requisito

**FR-001 — el total es el elemento de mayor jerarquía.** En el paso 3 el orden del DOM es: bloque de total → `<h1>` del método → bloque índigo → resumen → comprobante → enviar. La cifra `$ 72.500` en `text-3xl font-bold` es visiblemente el texto más grande de la vista, por encima del `<h1>` (`text-lg`). Confirmado en captura a 360 px.

**FR-016 — orden vertical.** Verificado con `compareDocumentPosition` en vivo y visualmente: el resumen queda **después** del bloque índigo de datos bancarios y **antes** de la zona de comprobante.

**FR-003 / FR-006 / RN-004 — un solo conteo.** Medido carácter por carácter en la pantalla real, en tres pedidos distintos:

| Pedido | Líneas | Unidades | Bloque de total | `<summary>` | ¿Idénticos? |
|--------|-------:|---------:|-----------------|-------------|-------------|
| Leo, con promoción | 6 | 8 | `8 productos` | `8 productos` | ✅ |
| Leo, ampliado | 11 | 13 | `13 productos` | `13 productos` | ✅ |
| Ana, sin promoción | 3 | 5 | `5 productos` | `5 productos` | ✅ |

Nótese que **líneas ≠ unidades** en los tres casos: es la distinción de FR-003 (unidades, lo que se muestra) frente a FR-008 (líneas, el umbral del colapso), y el componente no las mezcló.

**FR-008 — fronteras del umbral**, verificadas en la pantalla real: **3 líneas → llega expandido** (frontera inferior); **6 líneas → contraído**; **11 líneas → contraído**.

**FR-004 / SC-004 — el total coincide con el paso 1 y con lo que cobrará el cajero.** Con la promoción "test 25" vigente: el paso 1 mostró `$ 72.500`, el paso 3 mostró `$ 72.500`, y el pedido persistido en `heladeria.customer_orders` (tras ampliarlo a 11 líneas) quedó con un total de lista de **$169.500** y un **monto a cobrar de $145.500** — exactamente el total que el comensal vio en la pantalla de pago. El Ahorro de $24.000 es la diferencia.

**FR-012 — la fila "Ahorro".** Con promoción vigente: `Ahorro $ 24.000` en **los dos pasos**, en el mismo formato. Sin promoción (`discounted_total: null`): **ninguna fila "Ahorro"** en ninguno de los dos pasos, ni en cero ni vacía.

**FR-015 — adicionales cobrados una vez por línea** (spec 089). La línea `2× Granizado del diablo · Grande` con `Gomitas x2` mostró **$18.000** = precio de paquete $16.000 + Gomitas 2 × $1.000. Si los adicionales escalaran con la cantidad del producto habría dado $20.000. El resumen **muestra** ese número; no lo recompone.

**FR-010 — expandir no dispara nada.** Instrumentando `window.fetch` y `XMLHttpRequest.prototype.open` en la página y alternando el `<details>` **8 veces seguidas** (4 expandir + 4 contraer): **0 fetch, 0 XHR**, total sin cambios, ningún spinner.

**FR-019 / SC-009 — sin desborde horizontal a 320 px.** Con el resumen expandido y nombres/notas largas: `document.documentElement.scrollWidth === window.innerWidth === 320`, **cero elementos** con `getBoundingClientRect().right` fuera del viewport. Los nombres largos y la nota larga se recortan (`scrollWidth > clientWidth`) en vez de ensanchar la caja; los montos quedan completos y legibles.

**FR-020 — aplica a cualquier método que exija comprobante.** Recorridos **los dos** métodos no-efectivo, con los nombres literales:

| Método recorrido | Campos | ¿Total destacado? | ¿Resumen en la misma posición? |
|------------------|--------|-------------------|--------------------------------|
| **"Nequi"** | `celular` (texto) + `qr` (`format: 'image'`) | ✅ `$ 72.500` / `8 productos` | ✅ |
| **"Transferencia Bancolombia [092-test]"** | `cuenta` y `tipo_cuenta`, **sin** campo de imagen | ✅ `$ 72.500` / `8 productos` | ✅ |

El comportamiento no dependió de cómo el negocio nombró el método ni de si tiene QR — que es exactamente lo que FR-020 exige y lo que el segundo método existe para demostrar.

**FR-018 / SC-005 — el flujo de comprobante, de punta a punta.** Adjuntar → vista previa `blob:` y botón habilitado; reemplazar → la vista previa cambia; quitar → vuelve el estado vacío y **el botón queda deshabilitado**; readjuntar → habilitado; **enviar** → presign + subida real a R2 (`heladeria/comprobantes/5793cab5…png`) y llegada a la pantalla de confirmación ("¡Pedido enviado!"). El resumen y el total permanecieron intactos en cada paso.

**US3 esc. 3 — rehidratación.** Con un comprobante ya subido, recargando el navegador en el paso 3: la vista previa reaparece desde su URL pública **sin pedir el archivo de nuevo**, y el resumen aparece con los mismos datos (contraído, 11 líneas) y el total intacto. Ninguno interfiere con el otro.

> **Hallazgo lateral, no un defecto**: al sembrar a mano una URL de comprobante que no era una clave de comprobante válida, el backend respondió `El comprobante no pertenece a este negocio o no es válido`. Es la validación de integridad de referencias R2 de la **spec 088** haciendo su trabajo. Sirvió además para comprobar que la franja de error del paso 3 sigue conviviendo bien con el bloque de total y el resumen.

**FR-021 / SC-008 — los dos pasos no divergen.** Con el mismo pedido, el paso 1 y el paso 3 mostraron las **mismas 6 líneas en el mismo orden**, el mismo formato de montos, el mismo `Ahorro $ 24.000` y el mismo `Total $ 72.500`.

**FR-022 — el paso 1 no tiene control de colapso.** En el paso 1 no existe ningún `<details>`, ningún `<summary>` y ningún chevron: el resumen se pinta plano y siempre expandido.

**Paso 1 sin promoción, idéntico a `develop`.** Con la promoción revertida a vencida y un pedido de 3 líneas: el paso 1 muestra solo las líneas y la fila `Total $ 62.000`, **sin fila "Ahorro"** — exactamente como antes del cambio. Es la evidencia de que A-103 está confinada al caso con promoción vigente.

**FR-023 / research.md D7 — el carrito del menú público no cambió.** El panel "Mi pedido" del menú público conserva su propio markup —controles de cantidad `−/+`, enlace "Editar adicionales", desglose `$ 8.000 c/u + $ 1.000 adicionales`— y **no tiene ningún `<details>`**. No se migró, por decisión explícita, y se ve igual que antes.

---

## Accesibilidad (SC-006)

**PARCIAL. El anuncio con lector de pantalla real sigue PENDIENTE (T021)** — es el único criterio de la spec que queda sin verificar.

### Lo que sí quedó verificado (2026-10-02)

**El árbol de accesibilidad del navegador expone la semántica de *disclosure* y su estado.** Leyendo el a11y tree de Chrome sobre la pantalla real del paso 3:

```
DisclosureTriangle "Resumen del pedido 8 productos" expandable            ← contraído
DisclosureTriangle "Resumen del pedido 13 productos" expandable expanded  ← expandido
```

Es decir: el rol es de *disclosure*, el **nombre accesible** incluye el título y el conteo, y el **estado** (`expanded`) aparece y desaparece según el `<details>`. Ocultar el marcador nativo con `list-none` + `[&::-webkit-details-marker]:hidden` **no** costó la semántica, que era el riesgo que research.md D1 señalaba.

**Operable sin JavaScript propio**: activar el `<summary>` alterna el panel, y el componente no añadió `tabindex`, `role`, `aria-expanded` ni ningún `keydown` — así que no hay forma de haberlos escrito mal.

### Lo que falta y por qué importa (T021)

Que Chrome lo exponga en su árbol **no garantiza** que un lector lo pronuncie. Falta el recorrido con lector real, **empezando por Safari + VoiceOver en iPhone**, que es la plataforma objetivo del comensal y el único entorno donde `display:flex` sobre el `<summary>` puede costar la semántica nativa en la que SC-006 se apoya por completo (research.md D1). Después TalkBack; NVDA u Orca solo como contraste de escritorio.

**Empezar por Safari + VoiceOver en iPhone**, que es la plataforma objetivo real del comensal y el único entorno donde `display:flex` sobre el `<summary>` puede costar la semántica nativa de *disclosure* en la que SC-006 se apoya por completo (research.md D1). Después TalkBack; NVDA u Orca solo como contraste de escritorio.

| Lector | Versión | Paso | Frase literal anunciada | Resultado |
|--------|---------|------|-------------------------|-----------|
| — | — | Navegar hasta la sección (nombre + estado) | pendiente | pendiente |
| — | — | Activar con `Enter` (cambio de estado) | pendiente | pendiente |
| — | — | Activar con `Espacio` | pendiente | pendiente |

Registrar **la frase literal** de cada paso, no "funciona".

**Lo que sí está verificado por construcción** (research.md D1): el `<summary>` es focusable por defecto y responde a `Enter` y `Espacio` sin una sola línea de JavaScript; el estado lo lleva el atributo `open` del `<details>`, no el chevron, así que ocultar el marcador nativo no quita la semántica. No se añadió `tabindex`, `role`, `aria-expanded` ni ningún `keydown` — y por lo tanto no hay forma de haberlos escrito mal.

**T021-b** (palancas si el estado no se anuncia en algún lector): **no ejecutada**, porque depende del resultado de T021. Orden previsto: (a) sacar el `flex` del `<summary>` a un `<span>` hijo; (b) `aria-expanded` sincronizado con el evento nativo `(toggle)` —contradice el contrato, así que exigiría corregirlo en el mismo commit—; (c) último recurso, texto visible "Ver" / "Ocultar".

---

## Cierre (quickstart.md §9)

- [X] `npm test` en `pos-heladeria`: 1.311 tests, **1.291 pasan**, 19 fallan — los **mismos 19 preexistentes** de `develop`, en seis archivos que esta spec no toca. Cero regresiones, +63 tests nuevos todos en verde.
- [X] Bloques **2, 3, 6, 7 y 8** de `quickstart.md` completos (US1, US2, comprobante, paso 1 vs. paso 3, fuera del checkout).
- [X] Medición del bloque 4 registrada con cifra real: **1,2** en el caso de SC-003. **Ninguna palanca de RN-002 aplicada** — no hizo falta (T023 no aplica).
- [ ] ⚠️ Bloque **5 incompleto**: falta el recorrido con **lector de pantalla real** (T021). El árbol de accesibilidad de Chrome sí expone rol de *disclosure* y estado `expanded`, pero eso no sustituye oír al lector.
- [X] Decisión **D6 confirmada** con el negocio el 2026-10-02 y registrada en los tres lugares que correspondía: **A-103** en el registro de anomalías, la corrección de `spec.md` → *Impacto sobre Funcionalidades Existentes*, y este archivo.
- [X] Datos de prueba limpiados o marcados: promoción revertida a sus valores originales, método `[092-test]` desactivado, pedido de verificación identificable.

### Lo que queda pendiente, en una línea

**Solo T021** (y, condicionado a él, T021-b): el anuncio del estado del colapsable con un lector de pantalla real, empezando por Safari + VoiceOver en iPhone. Es lo único de la spec que una máquina no puede verificar por sí sola.

---

## Limitaciones

1. **Quedan dos fuentes de markup del resumen, no una.** El componente compartido del checkout (`checkout-order-summary.component.ts`, consumido por los pasos 1 y 3) y el carrito del menú público (`modules/tables/components/cart.component.ts`), que pinta el mismo formato de líneas y Total y **no se migró** por decisión explícita (research.md D7): FR-021 nombra solo los dos pasos del checkout y *Fuera de Alcance* prohíbe mostrar el resumen en pantallas no nombradas. RN-004 queda satisfecha en su letra —los dos pasos del checkout ya no pueden divergir—, pero el riesgo de divergencia con el carrito del menú sigue vivo. **Candidato natural a un spec propio de limpieza**; aquí queda anotado, no hecho.

2. **Sin impuestos ni subtotal, a propósito.** Mientras el pedido del comensal no los calcule en ningún nivel, ninguna pantalla los muestra (FR-013, RN-003). No es un pendiente: es la decisión de no inventar información frente al comensal. Si algún día el pedido llegara a exponerlos, esta spec no bloquea añadir la fila — solo prohíbe inventarla mientras no exista.

3. **El resumen no se recarga en vivo.** Refleja el pedido tal como se cargó al entrar al checkout (*Assumptions* y *Fuera de Alcance*). Por eso el estado inicial del `<details>` se congela en `ngOnInit` (research.md D2): hoy el carrito no muta en el paso 3, pero dejar `[attr.open]` enlazado a una expresión viva sería dejar armado un fallo que aparecería el día que alguien añada recarga del carrito a esa pantalla — Angular reescribiría el atributo y cerraría de golpe el panel que el comensal acaba de abrir (FR-009).

4. **El componente compartido no tiene salidas, y eso es el contrato.** Es lo que hace que FR-010 no se pueda violar ni por accidente ni por un cambio futuro descuidado. Añadirle un `@Output` más adelante (p. ej. "editar línea") es un cambio de alcance que necesita su propio spec — *Fuera de Alcance* ya prohíbe la edición desde esta pantalla.
