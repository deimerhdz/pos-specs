# Quickstart de Validación: Pestañas para Ver el Pedido Pagado Junto al Pago Pendiente de la Misma Mesa

Guía para verificar manualmente, de punta a punta, que el fix cumple los criterios de aceptación
del [spec.md](./spec.md) una vez implementado. No sustituye los tests automatizados (ver
[research.md §3](./research.md)).

## Prerrequisitos

- Repositorio `../pos-heladeria` con las dependencias instaladas.
- Backend `../pos-backend` corriendo localmente.
- Un turno de caja abierto, una mesa con un pedido ya cobrado por adelantado (`checkout_and_send`,
  spec 035/047), y la posibilidad de generar un segundo pedido QR en la misma mesa (misma sesión de
  comensal) que quede pendiente de confirmar el pago (efectivo o transferencia).

## Arrancar el entorno

```bash
cd ../pos-heladeria
pnpm start   # o el comando de desarrollo configurado en package.json
```

Abrir la Terminal de Mesas (`table-sessions` / ruta del módulo `tables`) en el navegador.

## Escenarios a validar (mapeados a los Acceptance Scenarios del spec)

1. **Preparar el estado**: cobrar por adelantado un pedido de mostrador en una mesa (queda
   `'pagada'`). Desde la misma sesión de mesa, generar un segundo pedido por QR que quede pendiente
   de confirmar el pago (efectivo o transferencia).
2. **Aparecen las dos pestañas**: seleccionar esa mesa y confirmar que el encabezado del panel
   central muestra dos pestañas: "🔔 Pagos por confirmar" y "Pedido de la mesa" — no el texto plano
   de siempre.
3. **Pestaña activa por defecto**: confirmar que "Pagos por confirmar" está resaltada/activa al
   entrar, y que el contenido mostrado es el bloque de confirmación de pago de siempre.
4. **Cambiar de pestaña**: pulsar "Pedido de la mesa" y confirmar que el panel muestra el pedido ya
   pagado con su resumen completo (ítems, total, "Imprimir Factura", "Liberar Mesa" en el panel
   derecho) — el mismo panel de siempre, no un resumen aparte.
5. **Volver a la otra pestaña**: pulsar "Pagos por confirmar" y confirmar que se vuelve a ver el
   bloque de confirmación, sin haber perdido nada del pedido pagado.
6. **Confirmar el pago pendiente**: desde la pestaña "Pagos por confirmar", confirmar el efectivo o
   aprobar el comprobante. Confirmar que, si ya no queda ningún pago pendiente de esa mesa, las
   pestañas desaparecen y el panel central vuelve a mostrar directamente el pedido (ahora sin
   pestañas), igual que antes de que existiera el pago pendiente.
7. **Casos sin pestañas** (sin cambios, deben seguir igual que hoy): seleccionar una mesa con
   **solo** un pago pendiente de confirmar (sin ningún pedido pagado/activo) y confirmar que no
   aparece ninguna pestaña — se ve directo el bloque de confirmación. Repetir con una mesa que
   **solo** tenga pedidos pagados/activos (sin nada pendiente) y confirmar que tampoco aparecen
   pestañas — se ve directo el pedido.

## Verificación automatizada

```bash
cd ../pos-heladeria
pnpm test -- pos-terminal.store       # existente + los casos nuevos de research.md §3
pnpm test -- table-sessions.component # existente + el bloque nuevo de pestañas
```

(Ajustar el runner exacto —`pnpm test`, `ng test`, `npx vitest`— al script real configurado en
`package.json` del proyecto en el momento de implementar; hoy es `ng test`.)
