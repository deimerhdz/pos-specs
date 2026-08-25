# Quickstart de Validación: Rediseño de Layout de la Terminal de Mesas

Guía para verificar manualmente, de punta a punta, que el rediseño cumple los criterios de aceptación
del [spec.md](./spec.md) una vez implementado. No sustituye los tests automatizados (ver
[research.md §5](./research.md) y [contracts/ui-store-contract.md](./contracts/ui-store-contract.md)).

## Prerrequisitos

- Repositorio `../pos-heladeria` con las dependencias instaladas (`pnpm install` o equivalente según el
  gestor del proyecto).
- Backend `../pos-backend` corriendo localmente (esta spec no requiere cambios de backend, pero la
  Terminal de Mesas necesita datos reales de mesas/catálogo para probarse).
- Al menos: 2-3 mesas con órdenes activas en distintos estados (una "Por confirmar" con pago QR
  pendiente, una "En preparación"), y suficientes mesas/pedidos para que la franja superior no quepa en
  el ancho de una pantalla estándar (para poder probar el carrusel).

## Arrancar el entorno

```bash
cd ../pos-heladeria
pnpm start   # o el comando de desarrollo configurado en package.json
```

Abrir la Terminal de Mesas (`table-sessions` / ruta del módulo `tables`) en el navegador.

## Escenarios a validar (mapeados a Acceptance Scenarios del spec)

1. **Franja superior con filtros** (Historia 1): confirmar que el panel de mesas ya no es una grilla
   agrupada por estado, sino una franja horizontal en la parte superior con las pestañas "Todas las
   Órdenes" / "Domicilios" / "Mesas". Alternar entre pestañas y verificar: "Mesas" y "Todas las Órdenes"
   muestran el mismo conjunto (no hay órdenes de Domicilio todavía); "Domicilios" muestra un listado
   vacío con mensaje claro, no un error.
2. **Carrusel**: con más tarjetas que las que caben en el ancho de pantalla, pulsar la flecha derecha y
   confirmar que el listado se desplaza un tramo (no una tarjeta) y revela tarjetas nuevas; llegar al
   final y confirmar que la flecha derecha se deshabilita; volver con la flecha izquierda hasta el
   inicio y confirmar que esa flecha se deshabilita ahí. Cambiar de pestaña de filtro estando
   desplazado y confirmar que el listado vuelve al inicio.
3. **Insignia + barra de tiempo conviven**: abrir una tarjeta de una mesa con pago QR pendiente y
   confirmar que muestra tanto la insignia "Por confirmar" (texto + color) como la barra de tiempo
   transcurrido, sin que una reemplace a la otra.
4. **Selección de tarjeta**: hacer clic en una tarjeta de mesa y confirmar que abre exactamente el mismo
   panel central/derecho que hoy se abre al seleccionar esa mesa (bloque de validación QR, constructor
   manual, o resumen de cuenta, según corresponda) — sin diferencias de comportamiento respecto a antes
   del rediseño.
5. **Menú central embebido** (Historia 2): con una orden manual en construcción, confirmar que el grid
   de productos está permanentemente visible en el panel central (no aparece como overlay de pantalla
   completa). Escribir un texto en el buscador y confirmar que el grid se filtra por nombre en tiempo
   real; seleccionar una categoría y confirmar que se filtra por categoría; combinar ambos filtros y
   confirmar que se intersectan; probar una combinación sin resultados y confirmar que aparece un estado
   vacío claro.
6. **Agregar producto sin overlay**: agregar un producto desde el grid central y confirmar que aparece
   en el resumen del panel derecho con la misma mecánica de cantidad/nota/eliminación ya existente.
7. **Orden QR de solo lectura**: abrir una mesa con orden QR en modo "Resumen de Cuenta" y confirmar que
   el grid de agregar productos no se ofrece para esa orden (igual que hoy).
8. **Toggle de sidebar global** (Historia 3): con el menú de navegación global visible, pulsar el botón
   en la parte superior de la Terminal de Mesas y confirmar que el sidebar se colapsa y el contenido
   ocupa el espacio liberado — en una pantalla de escritorio, no solo en móvil. Pulsar de nuevo y
   confirmar que vuelve a expandirse. Navegar a otra sección del panel administrativo con el sidebar
   colapsado y confirmar que las demás opciones de navegación siguen siendo accesibles.
9. **Impuesto en $0**: confirmar que el resumen de pago muestra una línea "Impuesto" con valor $0, sin
   campo editable, para cualquier orden.
10. **Sin regresión de comportamiento**: ejecutar al menos un ciclo completo de validación de pago QR
    (confirmar/rechazar comprobante) y un cobro manual completo (cobrar, facturar, enviar a cocina), y
    confirmar que ambos se comportan exactamente igual que antes del rediseño (spec 028) — solo cambia
    la disposición visual del panel derecho, no su lógica.

## Verificación automatizada

```bash
cd ../pos-heladeria
pnpm test -- pos-tables-panel      # nuevo spec, debe cubrir filtros + carrusel
pnpm test -- pos-checkout-panel    # spec existente, debe seguir en verde sin cambios de intención
pnpm test -- pos-terminal.store    # spec existente, debe seguir en verde
```

(Ajustar el runner exacto —`pnpm test`, `ng test`, `npx vitest`— al script real configurado en
`package.json` del proyecto en el momento de implementar.)
