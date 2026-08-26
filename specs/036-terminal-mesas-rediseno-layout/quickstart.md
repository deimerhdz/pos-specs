# Quickstart de Validación: Rediseño de Layout de la Terminal de Mesas

Guía para verificar manualmente, de punta a punta, que el rediseño cumple los criterios de aceptación
del [spec.md](./spec.md) una vez implementado. No sustituye los tests automatizados (ver
[research.md §6](./research.md) y [contracts/ui-store-contract.md](./contracts/ui-store-contract.md)).

## Prerrequisitos

- Repositorio `../pos-heladeria` con las dependencias instaladas.
- Backend `../pos-backend` corriendo localmente (esta spec no requiere cambios de backend, pero la
  Terminal de Mesas necesita datos reales de mesas/catálogo para probarse).
- Al menos: una mesa con pago QR en efectivo pendiente de confirmar, una mesa con transferencia ya
  aprobada, una mesa "En preparación" con un pedido manual en construcción, y una mesa libre.

## Arrancar el entorno

```bash
cd ../pos-heladeria
pnpm start   # o el comando de desarrollo configurado en package.json
```

Abrir la Terminal de Mesas (`table-sessions` / ruta del módulo `tables`) en el navegador.

## Escenarios a validar (mapeados a Acceptance Scenarios del spec)

1. **Pestañas de tipo + filtros de ocupación** (Historia 1): confirmar que la grilla de mesas muestra
   las pestañas "Mesas"/"Domicilios"/"Para llevar" arriba, y que "Mesas" conserva sin cambios los
   filtros de ocupación "Todas"/"Libres"/"Ocupadas"/"Pendientes" ya existentes. Seleccionar
   "Domicilios" o "Para llevar" y confirmar un listado vacío con mensaje claro (no un error).
2. **"Pagos por confirmar"**: con una mesa con efectivo pendiente y otra con transferencia aprobada,
   confirmar que ambas aparecen en la sección "Pagos por confirmar" debajo de la grilla, con el mismo
   dato (cliente/mesa, método, estado, total) que hoy se ve dentro del panel de cada mesa. Pulsar
   "Confirmar efectivo" desde esa sección y verificar que el pedido pasa a confirmado exactamente igual
   que al confirmarlo desde el panel de la mesa (revisar que la insignia de la mesa en la grilla también
   se actualiza, sin recargar la página).
3. **Selección desde cualquiera de las dos vistas**: hacer clic en una tarjeta de mesa desde la grilla y,
   por separado, desde "Pagos por confirmar", y confirmar en ambos casos que abre el mismo panel
   central/derecho que hoy se abre al seleccionar esa mesa (bloque de validación QR, lista de ítems del
   pedido, o resumen de cuenta, según corresponda).
4. **Mesa libre**: seleccionar una mesa libre y confirmar que sigue disponible la misma vía ya
   implementada para iniciar una orden sobre ella (spec 028, Historia 2), solo reubicada visualmente.
5. **Panel central: lista de ítems + "+ Agregar producto"** (Historia 2): abrir una mesa con pedido en
   construcción y confirmar que el panel central muestra por defecto la lista de ítems ya agregados
   ("Pedido de la mesa"), no una grilla de catálogo. Pulsar "+ Agregar producto" y confirmar que, dentro
   del mismo panel (sin overlay de pantalla completa), aparece la grilla con buscador por nombre y
   pestañas de categoría. Escribir un texto en el buscador y confirmar que la grilla se filtra por
   nombre en tiempo real; seleccionar una categoría y confirmar que se filtra por categoría; combinar
   ambos filtros y confirmar que se intersectan; probar una combinación sin resultados y confirmar que
   aparece un estado vacío claro.
6. **Agregar producto sin overlay**: seleccionar un producto desde la grilla y confirmar que el panel
   central regresa a la lista de ítems del pedido (mostrando el producto agregado) y que el resumen del
   panel derecho refleja la misma mecánica de cantidad/nota/eliminación ya existente.
7. **Orden QR de solo lectura**: abrir una mesa con orden QR en modo "Resumen de Cuenta" y confirmar que
   la lista de ítems se ve de solo lectura y que no se ofrece el botón "+ Agregar producto" para esa
   orden (igual que hoy el catálogo no está disponible ahí).
8. **Toggle de sidebar global** (Historia 3): con el menú de navegación global visible, pulsar el botón
   en la parte superior de la Terminal de Mesas y confirmar que el sidebar se colapsa y el contenido
   ocupa el espacio liberado — en una pantalla de escritorio, no solo en móvil. Pulsar de nuevo y
   confirmar que vuelve a expandirse. Navegar a otra sección del panel administrativo con el sidebar
   colapsado y confirmar que las demás opciones de navegación siguen siendo accesibles.
9. **Impuesto en $0**: confirmar que el resumen de pago muestra una línea "Impuesto" con valor $0, sin
   campo editable, para cualquier orden.
10. **"Dividir la cuenta" y "Facturar a nombre de" sin regresión**: en una mesa con varios comensales,
    pulsar "Dividir la cuenta entre varias personas" y confirmar que abre exactamente el mismo flujo de
    asignación por comensal que ya existe hoy (spec 010) — solo con la disposición visual ajustada.
    Confirmar que "Facturar a nombre de" sigue siendo el mismo campo de cliente ya existente (spec 011).
11. **Sin regresión de comportamiento**: ejecutar al menos un ciclo completo de validación de pago QR
    (confirmar/rechazar comprobante) y un cobro manual completo (cobrar, facturar, enviar a cocina), y
    confirmar que ambos se comportan exactamente igual que antes del rediseño (spec 028) — solo cambia
    la disposición visual, no la lógica.

## Verificación automatizada

```bash
cd ../pos-heladeria
pnpm test -- pos-tables-panel        # nuevo spec, debe cubrir pestañas de tipo + filtro de ocupación
pnpm test -- pos-terminal.store      # existente + casos nuevos para pendingPaymentsView/catalogProductsFiltered
pnpm test -- pos-checkout-panel      # spec existente, debe seguir en verde sin cambios de intención
pnpm test -- pos-order-panel         # spec existente, debe seguir en verde
pnpm test -- split-bill-panel        # spec existente, debe seguir en verde sin cambios de intención
pnpm test -- session-bill-panel      # spec existente, debe seguir en verde sin cambios de intención
```

(Ajustar el runner exacto —`pnpm test`, `ng test`, `npx vitest`— al script real configurado en
`package.json` del proyecto en el momento de implementar.)
