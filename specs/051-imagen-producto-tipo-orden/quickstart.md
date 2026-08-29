# Quickstart: Validación manual — Imagen de producto en el catálogo y organización del tipo de orden

**Prerrequisitos**:
- `pos-heladeria` corriendo localmente contra un backend (`pos-backend`) con al menos un producto
  **con** `image_url` configurada y uno **sin** ella (ver `products-page.component.ts` /
  `product-form.component.ts` para configurar la imagen de un producto desde el módulo de
  Catálogo si hace falta).
- Al menos una mesa en estado "Libre" en el tenant de prueba.
- Sesión de staff con acceso a la Terminal de Mesas.

## Escenario 1 — Imagen en la tarjeta del catálogo (US1, FR-001, FR-002)

1. Desde la Terminal de Mesas, abrir la creación de orden manual sobre una mesa libre (ruta que
   monta `ManualOrderPageComponent`).
2. En la grilla de catálogo (columna izquierda), ubicar un producto con imagen configurada.
   **Esperado**: la tarjeta muestra la fotografía del producto, además de nombre y precio.
3. Ubicar un producto sin imagen configurada. **Esperado**: la tarjeta muestra un ícono neutro de
   "sin imagen" en el mismo espacio (sin imagen rota, sin espacio en blanco desalineado respecto a
   las demás tarjetas), y sigue mostrando nombre y precio con claridad.
4. Si el producto ubicado en el paso 2 tiene descuento activo, confirmar que el badge "🏷️ …" sigue
   visible sobre la imagen, sin quedar tapado por ella.

## Escenario 2 — Imagen en el detalle/opciones (US2, FR-003)

1. Desde la misma pantalla, tocar la tarjeta del producto con imagen configurada.
2. **Esperado**: se abre el modal de detalle/opciones (`ProductSelectComponent`) y muestra la misma
   imagen del producto en su encabezado.
3. Tocar la tarjeta de un producto sin imagen. **Esperado**: el modal se abre correctamente, sin
   imagen rota ni espacio reservado vacío.

## Escenario 3 — Título sobre el listado de mesas (US3, FR-004, FR-005)

1. En la misma pantalla, con la pestaña "En Mesa" seleccionada (única habilitada), observar el
   bloque superior.
2. **Esperado**: aparece un título propio inmediatamente encima del listado horizontal de mesas,
   visualmente distinguible del encabezado "Tipo de Orden" que agrupa las pestañas (p. ej. distinto
   tamaño/peso de fuente) — no deben leerse como un único encabezado compartido.
3. Confirmar que las pestañas "Para Llevar" y "Domicilio" siguen deshabilitadas y sin cambio de
   comportamiento (FR-006).

## Regresión rápida

- Ejecutar la suite de `manual-order-page.component.spec.ts` (`pos-heladeria`) y confirmar que los
  8 casos existentes siguen en verde sin modificarlos (research.md, "Resumen de impacto en tests
  existentes").
- Confirmar que agregar un producto al carrito desde una tarjeta con imagen sigue funcionando igual
  que hoy (mismo flujo de `store.openConfig(p)` → `store.addDraftFromSelection($event)`).
