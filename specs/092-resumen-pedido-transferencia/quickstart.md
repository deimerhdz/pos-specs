# Quickstart: verificar el resumen del pedido en la pantalla de pago

**Spec**: [spec.md](./spec.md) | **Plan**: [plan.md](./plan.md) | **Contrato**: [contracts/checkout-order-summary.md](./contracts/checkout-order-summary.md)

Guía de **validación**, no de implementación. Cada bloque dice qué historia, FR y criterio de éxito prueba. Los bloques 4, 5 y 6 **no se pueden sustituir por tests automáticos**: jsdom no hace layout ni habla con un lector de pantalla (research.md D10).

## 0. Prerrequisitos

- `pos-heladeria` en la rama `feat/092-payment-step-order-summary` (creada desde `develop`, Principio XIV). **`pos-backend` no se toca**: basta con el backend de desarrollo en `develop` corriendo en `http://localhost:8000`.
- Postgres de desarrollo con un negocio de prueba, su menú sembrado y al menos una mesa con QR activo.
- **Al menos un método de pago no-efectivo** configurado y activo, con datos de texto; idealmente **dos**: uno con campo `format: 'image'` (QR) y otro sin él, para cubrir dos casos límite de la spec.
- **Una promoción vigente** que cubra alguna variante del menú, para poder ver la fila "Ahorro" (FR-012). Sin promoción activa, los escenarios de Ahorro no son verificables.
- Frontend: `npm start` en `pos-heladeria`.
- Navegador con DevTools (emulación de dispositivo) y un lector de pantalla disponible (VoiceOver en iOS/macOS, TalkBack en Android, o NVDA/Orca en escritorio).

## 1. Pruebas automáticas

Todo se ejecuta en `pos-heladeria`. **No hay nada que correr en `pos-backend`** — ningún archivo suyo cambia; sus characterization tests (`app/characterization_tests/test_cart_service.py`, que fijan `discounted_total < total` y `None` sin promoción) siguen siendo el contrato del que sale el "Ahorro" y deben quedar intactos.

```bash
cd pos-heladeria
npm test
```

**Esperado**: suite completa en verde. Los archivos que esta spec toca o crea:

| Archivo | Qué cubre |
|---------|-----------|
| `src/app/modules/tables/services/dining-cart.service.spec.ts` | `grossTotal` y `savings` con `discounted_total` **ausente**, **`null`** y **menor que `total`** → `savings` debe dar `0`, `0` y la diferencia exacta (research.md D5) |
| `src/app/modules/tables/pages/checkout/checkout-order-summary.component.spec.ts` | FR-003 a FR-009, FR-012 a FR-015, FR-021, FR-022 (ver detalle abajo) |
| `src/app/modules/tables/pages/checkout/review-step.component.spec.ts` | Base de no-regresión del paso 1: formato de línea, orden y Total. **Se escribe antes de la extracción**, no después |
| `src/app/modules/tables/pages/checkout/transfer-details-step.component.spec.ts` | Bloque de total y ubicación del resumen. **Atención**: su doble de `DiningCartService` es hoy `{ clear, clearDiner }` y hay que ampliarlo con `lines`, `total`, `count` y `savings` o **todo el archivo se cae** (research.md D10) |

Casos que la suite del componente compartido debe contener, uno por uno:

- Pedido de **3 líneas** con `collapsible: true` → el `<details>` llega con `open` (FR-008, frontera inferior).
- Pedido de **4 líneas** con `collapsible: true` → el `<details>` llega **sin** `open` (FR-008, frontera superior). Ambos importan `UMBRAL_EXPANDIDO_POR_DEFECTO`, no el literal `3`.
- `collapsible: false` → **no existe** ningún `<details>` ni `<summary>` en el render (FR-022).
- Línea con adicionales y nota → aparecen cantidad, producto, presentación, adicionales, nota y total de línea (FR-007).
- Línea **sin** adicionales y **sin** nota → esos dos renglones no se pintan (FR-007).
- `savings: 1500` → aparece la fila "Ahorro" con `$ 1.500` (FR-012).
- `savings: 0` → **no** aparece ninguna fila "Ahorro", ni en cero ni vacía (FR-012).
- Cualquier combinación → el render **no contiene** las palabras "Impuesto", "IVA" ni "Subtotal" (FR-013, SC-007).
- `count: 5` con 2 líneas → el `<summary>` y el conteo del componente dicen **"5 productos"**, el mismo número y la misma palabra (FR-003, FR-006, RN-004).
- `count: 1` → **"1 producto"**, en singular (research.md D11).
- Con el `<details>` abierto a mano (`details.open = true`) → ni el carrito cambia ni se invoca ningún servicio; el componente **no tiene salidas** que disparar (FR-010). En jsdom se fija `details.open` directamente: el toggle por click del `<summary>` no es confiable allí (research.md D1).

## 2. Recorrido manual — Historia 1: ver cuánto debo transferir

1. Abrir el QR de la mesa en el navegador, identificarse como comensal.
2. Armar un pedido con **varios productos**, incluyendo uno con adicionales y una nota.
3. Anotar el **Total** del paso 1 ("Revisa tu pedido"). Continuar → elegir un método de pago **no-efectivo**.
4. En la pantalla de datos de pago, **sin tocar ni desplazar nada**:
   - [ ] El total a pagar es visible de inmediato (FR-002, SC-001).
   - [ ] Es el **texto más grande de la vista** y el primer dato bajo el encabezado, **por delante del nombre del método** (FR-001).
   - [ ] Lo acompaña el conteo de productos en unidades, p. ej. "5 productos" (FR-003).
   - [ ] Coincide **hasta el último dígito** con el total anotado en el paso 1 (SC-004).
5. Contraer la sección "Resumen del pedido" → [ ] el total sigue visible; nunca queda dentro de la zona colapsable (FR-002, US1 esc. 2).
6. Con la promoción vigente aplicada → [ ] el total mostrado es el **ya descontado**, no el de lista (FR-004).
7. [ ] El número y la palabra del conteo del bloque de total son **idénticos** a los del título del resumen — nunca dos conteos distintos (FR-006, US1 esc. 5).

## 3. Recorrido manual — Historia 2: revisar el pedido sin retroceder

Con el mismo pedido, en la pantalla de datos de pago:

1. [ ] Existe una sección titulada **"Resumen del pedido"** que indica en su propio título cuántos productos contiene (FR-005, FR-006).
2. Con un pedido de **3 líneas o menos** → [ ] llega **expandida** (FR-008).
3. Con un pedido de **más de 3 líneas** → [ ] llega **contraída**, y al abrirla se ven **todas** las líneas (FR-008).
4. Expandida, en una línea con adicionales y nota → [ ] se ven cantidad, producto, presentación, adicionales, nota y valor total de la línea (FR-007).
5. Con la promoción vigente → [ ] aparece la fila **"Ahorro"** con el descuento, además del Total (FR-012).
6. Quitar del pedido el producto cubierto por la promoción (o pausarla) y volver a entrar → [ ] **no** aparece ninguna fila "Ahorro", ni en cero ni vacía (FR-012).
7. [ ] No aparece **ninguna** fila de impuestos ni de subtotal (FR-013, SC-007).
8. Verificar la línea con varias unidades → [ ] el adicional se cobra **una sola vez**, no multiplicado por la cantidad; el valor de la línea coincide con lo que mostraba el paso 1 (FR-015, spec 089).

## 4. Verificación de layout — Historia 3, SC-003 *(no automatizable)*

DevTools → Toggle device toolbar → **360 × 640** (preset "Galaxy S" o dimensiones manuales).

1. Armar un pedido de **exactamente 6 líneas**. Llegar a la pantalla de datos de pago con un método que **tenga QR** (el caso más exigente).
2. [ ] El resumen llega **contraído** (6 > 3).
3. Medir en la consola de DevTools, con el resumen en su **estado inicial** (contraído) y sin haber desplazado nada:

   ```js
   document.documentElement.scrollHeight / window.innerHeight
   ```

   - **Esperado: ≤ 2,0** (FR-017, SC-003). A 360 × 640 px, con `window.innerHeight` ≈ 640, el umbral equivale a ≈ **1.280 px** de `scrollHeight`.
   - Anotar **el número real con un decimal**, nunca un "sí/no": es lo que permite comparar la medición entre personas y entre commits, y lo que convierte este criterio en reproducible.
   - Medir en **emulación de DevTools**, donde `innerHeight` es estable; en un teléfono real la barra de direcciones lo cambia al desplazar y la cifra deja de ser comparable.
   - **Si da más de 2,0**: no se cumple. Aplicar RN-002 (prevalece la acción de enviar) con las palancas de research.md D9, **en este orden**: (a) compactar el bloque de total poniendo la cifra y el conteo en la misma línea; (b) reducir el `max-width` del QR; (c) bajar el umbral de FR-008 — que es una **decisión de producto** y se registra, no se cambia por cuenta propia. Volver a medir y anotar la cifra después de cada palanca.
4. Repetir con un pedido de **10+ líneas** y el resumen expandido → [ ] la lista crece, el total sigue arriba y el botón sigue alcanzable al final (caso límite de la spec).
5. Repetir con el **segundo método no-efectivo**, sin QR y con otro nombre (solo datos de texto) → [ ] la jerarquía se mantiene, el resumen no cambia de posición y el total sigue destacado — el comportamiento no depende de cómo el negocio nombró el método (FR-020, caso límite de la spec).
6. Reducir el ancho a **320 px**, con el resumen expandido y nombres de producto y notas largas → [ ] no hay **desplazamiento horizontal** y nada queda recortado de forma ilegible (FR-019, SC-009).

**Registrar la medición del punto 3 en `implementation-notes.md`.** Es el único criterio de la spec cuyo cumplimiento no se puede afirmar sin medirlo.

## 5. Verificación de accesibilidad — SC-006, FR-011 *(no automatizable)*

Teclado, sin mouse:

1. Desde el inicio de la pantalla, avanzar con `Tab` hasta la sección del resumen → [ ] el título recibe foco visible.
2. `Enter` → [ ] se expande. `Enter` otra vez → [ ] se contrae. Repetir con `Espacio` → [ ] mismo resultado (FR-009, FR-011).
3. Expandir y contraer **varias veces seguidas** → [ ] el pedido no cambia, el total no cambia, no aparece ningún spinner y DevTools → Network **no registra ninguna petición nueva** (FR-010, US3 esc. 4).

Con lector de pantalla activo:

4. Navegar hasta la sección → [ ] se anuncia su nombre y su **estado** ("contraído" / "expandido", o el equivalente del lector).
5. Activarla → [ ] el cambio de estado se anuncia (SC-006).

## 6. No-regresión del flujo de comprobante — Historia 3, SC-005

En la pantalla de datos de pago, de punta a punta:

1. [ ] Adjuntar una imagen → se ve la vista previa.
2. [ ] "Cambiar imagen" por otra → la vista previa se reemplaza.
3. [ ] "Quitar" → vuelve al estado vacío y el botón "Enviar pedido" queda deshabilitado.
4. [ ] Adjuntar de nuevo y **"Enviar pedido"** → el pedido se crea y se llega al paso de confirmación, sin bloqueos (FR-018, SC-005).
5. **Rehidratación**: con un comprobante ya subido, recargar el navegador en el paso de datos de pago → [ ] la vista previa reaparece sin pedir el archivo de nuevo, **y** el resumen aparece con los mismos datos; ninguno interfiere con el otro (US3 esc. 3, FR-018).
6. [ ] Los botones de **copiar** campo y **descargar** QR siguen funcionando igual (FR-018, specs 060/088).
7. [ ] **Volver** y **salir sin enviar** siguen funcionando y no crean ningún pedido (FR-018).

## 7. No-regresión del paso de revisión — Historia 4, SC-008

1. Con el **mismo** pedido, abrir el paso 1 y el paso 3 y compararlos lado a lado:
   - [ ] Líneas, orden, formato de los montos, Ahorro y Total son **idénticos** (FR-021, US4 esc. 1, SC-008).
2. En el paso 1 → [ ] el resumen está **siempre expandido**, sin chevron y sin control de colapso (FR-022, US4 esc. 2).
3. ⚠️ **Punto de decisión pendiente** (research.md D6): con una promoción vigente, el paso 1 ahora muestra la fila "Ahorro" que **antes no mostraba**. Confirmar con el negocio **antes** de cerrar esta historia. Si la acepta → registrar **A-103** en `specs/000-reconocimiento/registro-de-anomalias.md` y corregir la frase de "Impacto sobre Funcionalidades Existentes" de `spec.md`. Si no → el paso 1 pasa `showSavings: false` y la excepción a US4 esc. 1 se registra en la spec.
4. Sin promoción vigente → [ ] el paso 1 se ve **exactamente** igual que antes del cambio (comparar contra `develop`).

## 8. No-regresión fuera del checkout — FR-023

1. [ ] El carrito del menú público (botón del carrito, antes de entrar al checkout) sigue pintando sus líneas y su total igual que antes — **no se migró** al componente compartido, por decisión explícita (research.md D7).
2. [ ] En la terminal del cajero, el importe al **confirmar el pago** de ese pedido coincide con el total que vio el comensal (SC-004, FR-023).
3. [ ] Ninguna pantalla del cajero, del mesero ni del administrador cambió (FR-023).
4. [ ] El flujo de pago en **efectivo** no pasa por esta pantalla y sigue enviando el pedido directamente desde el paso de método (fuera de alcance).

## 9. Cierre

- [ ] `npm test` en verde en `pos-heladeria`.
- [ ] Bloques 2, 3, 6, 7 y 8 completos.
- [ ] Medición del bloque 4 registrada en `implementation-notes.md`, con el resultado real y, si hubo ajuste, qué palanca de RN-002 se usó.
- [ ] Bloque 5 completado con un lector de pantalla real, no asumido.
- [ ] Decisión D6 confirmada con el negocio y registrada donde corresponda.
- [ ] Datos de prueba creados para esta verificación limpiados o marcados (`[092-test]`) en la base de datos de desarrollo.
