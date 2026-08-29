# Quickstart: Validación manual — Selector de mesa buscable

**Prerrequisitos**:
- `pos-heladeria` corriendo localmente contra un backend (`pos-backend`) con al menos 5-8 mesas
  configuradas para el tenant de prueba, con al menos una ocupada.
- Sesión de staff con acceso a la Terminal de Mesas.

## Escenario 1 — Seleccionar una mesa por búsqueda (US1, FR-001, FR-002, FR-005)

1. Desde la Terminal de Mesas, abrir la creación de orden manual sobre una mesa libre.
2. En el panel derecho, ubicar el bloque "Mesas". **Esperado**: ya no es una rejilla de botones,
   sino un control de select (botón con el nombre de la mesa actual + flecha).
3. Hacer clic sobre el select. **Esperado**: se abre un listado con un campo de búsqueda arriba y
   todas las mesas del tenant debajo.
4. Escribir parte del nombre de otra mesa libre (p. ej. "5" para "Mesa 5"). **Esperado**: el
   listado se filtra a las mesas coincidentes.
5. Hacer clic sobre la mesa filtrada. **Esperado**: el select se cierra, muestra la mesa elegida, y
   la sesión de armado de pedido pasa a esa mesa (igual que antes con la rejilla).

## Escenario 2 — Nombre y estado visibles; mesas ocupadas no seleccionables (US2, FR-003, FR-004)

1. Abrir el select de mesas nuevamente.
2. **Esperado**: cada fila del listado muestra el nombre de la mesa y su estado (p. ej.
   "Mesa 3 — Libre", "Mesa 5 — Ocupada").
3. Intentar hacer clic sobre una mesa ocupada (que no sea la ya seleccionada). **Esperado**: no
   pasa nada — la mesa sigue visible en el listado, pero no se selecciona ni cierra el select.
4. Confirmar que la mesa actualmente seleccionada sí se puede volver a elegir desde el listado sin
   error.

## Regresión rápida

- Ejecutar la suite de `manual-order-page.component.spec.ts` y `searchable-select.component.spec.ts`
  (`pos-heladeria`) y confirmar que todos los casos existentes y nuevos están en verde.
- Confirmar que el resto de la pantalla (tipo de orden, catálogo con imágenes, carrito, totales,
  confirmar) sigue funcionando igual que antes de esta spec.
