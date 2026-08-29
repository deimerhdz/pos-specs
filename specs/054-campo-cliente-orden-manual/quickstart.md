# Quickstart: Validación manual — Campo "Cliente" en la creación de orden manual

**Prerrequisitos**:
- `pos-heladeria` corriendo localmente contra un backend (`pos-backend`) con al menos una mesa
  libre para el tenant de prueba.
- Sesión de staff con acceso a la Terminal de Mesas.

## Escenario 1 — Valor por defecto visible (US1, FR-001, FR-002)

1. Desde la Terminal de Mesas, abrir la creación de orden manual sobre una mesa libre.
2. En el panel derecho, bajo el bloque "Mesas", observar el campo "Cliente".
3. **Esperado**: ya muestra "Consumidor final", en modo de solo lectura (no se puede escribir
   directamente sobre él).

## Escenario 2 — Editar el nombre del cliente (US2, FR-003, FR-004, FR-005)

1. Hacer clic en el ícono de edición (✏️) junto al campo "Cliente".
2. **Esperado**: el campo se vuelve editable.
3. Borrar "Consumidor final" y escribir un nombre, p. ej. "María Pérez".
4. Hacer clic fuera del campo. **Esperado**: el campo vuelve a modo solo lectura y muestra "María
   Pérez".
5. Repetir el proceso, pero esta vez borrar todo el texto sin escribir nada y hacer clic fuera.
   **Esperado**: el campo vuelve a mostrar "Consumidor final" (nunca queda vacío).

## Escenario 3 — El nombre queda guardado en la orden creada (US3, FR-006)

1. Con el campo "Cliente" en "Consumidor final" (sin editar), agregar un producto al carrito y
   confirmar y enviar la orden.
2. **Esperado**: la orden se crea correctamente; al revisarla (p. ej. desde la Terminal de Mesas o
   el detalle de la orden), su cliente es "Consumidor final".
3. Repetir el flujo completo, esta vez editando el campo "Cliente" a un nombre específico antes de
   confirmar. **Esperado**: la orden creada queda con ese nombre específico como cliente.

## Regresión rápida

- Ejecutar la suite de `manual-order-page.component.spec.ts` (`pos-heladeria`) y confirmar que
  todos los casos existentes y nuevos están en verde.
- Confirmar que el resto de la pantalla (tipo de orden, selector de mesa, catálogo con imágenes,
  carrito, totales, confirmar) sigue funcionando igual que antes de esta spec.
