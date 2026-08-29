# Quickstart: Validación manual — Panel derecho unificado en la creación de orden manual

**Prerrequisitos**:
- `pos-heladeria` corriendo localmente contra un backend (`pos-backend`) con al menos 5-8 mesas
  configuradas para el tenant de prueba (para validar la rejilla de mesas con varias filas).
- Sesión de staff con acceso a la Terminal de Mesas.

## Escenario 1 — Todo el flujo de configuración en el panel derecho (US1, FR-001 a FR-006)

1. Desde la Terminal de Mesas, abrir la creación de orden manual sobre una mesa libre.
2. **Esperado**: la barra superior solo muestra "← Volver a la Terminal". Ni "Tipo de Orden" ni el
   listado de mesas aparecen ahí.
3. Observar el panel derecho, de arriba hacia abajo. **Esperado**: "Tipo de Orden" con sus tres
   pestañas, luego "Mesas" con el listado de mesas en rejilla, luego "Nueva orden" con el carrito
   vacío, el resumen de totales y "Confirmar y Enviar".
4. Elegir una mesa libre distinta desde el panel derecho. **Esperado**: mismo comportamiento que
   antes (mesa se resalta, mesas ocupadas siguen deshabilitadas), sin salir del panel derecho.
5. Confirmar que "Para Llevar" y "Domicilio" siguen deshabilitadas, con el mismo tooltip que antes.
6. Desde el catálogo (izquierda), agregar un producto al carrito. **Esperado**: aparece en la
   sección de carrito del panel derecho, sin haber tenido que mover el mouse a la barra superior en
   ningún momento de los pasos 4-6.

## Escenario 2 — Ancho del panel derecho (US2, FR-007, SC-002)

1. Con la pantalla en un tamaño de escritorio estándar (≥1280px de ancho de ventana), medir/observar
   el ancho del panel derecho.
2. **Esperado**: visiblemente más ancho que antes de esta spec (320px → 400px); ninguna de las tres
   secciones ("Tipo de Orden", "Mesas", "Nueva orden") se ve recortada ni requiere scroll horizontal
   interno.
3. Confirmar que el catálogo de productos (izquierda) sigue siendo usable, sin tarjetas de producto
   amontonadas.

## Escenario 3 — Todas las mesas visibles en la rejilla (SC-003)

1. Con un tenant que tenga más de 4 mesas configuradas, observar la sección "Mesas" del panel
   derecho.
2. **Esperado**: todas las mesas configuradas son visibles (en varias filas si hace falta), sin
   scroll horizontal y sin ninguna oculta.

## Regresión rápida

- Ejecutar la suite de `manual-order-page.component.spec.ts` (`pos-heladeria`) y confirmar que los
  13 casos existentes (8 originales + 5 de spec 051) siguen en verde sin modificar su intención
  (research.md, "Resumen de impacto en tests existentes").
- Confirmar que agregar un producto, cambiar de mesa y confirmar el pedido siguen funcionando igual
  que antes de esta spec.
