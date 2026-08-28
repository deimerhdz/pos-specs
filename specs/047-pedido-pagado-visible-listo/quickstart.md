# Quickstart de Validación: Pedido de Mostrador Pagado Sigue Visible Hasta Liberar la Mesa

Guía para verificar manualmente, de punta a punta, que el fix cumple los criterios de aceptación
del [spec.md](./spec.md) una vez implementado. No sustituye los tests automatizados (ver
[research.md §4](./research.md)).

## Prerrequisitos

- Repositorio `../pos-heladeria` con las dependencias instaladas.
- Backend `../pos-backend` corriendo localmente (esta spec no requiere cambios de backend, pero la
  Terminal de Mesas necesita datos reales de mesas/catálogo/turno de caja para probarse).
- Un turno de caja abierto y al menos una mesa libre.

## Arrancar el entorno

```bash
cd ../pos-heladeria
pnpm start   # o el comando de desarrollo configurado en package.json
```

Abrir la Terminal de Mesas (`table-sessions` / ruta del módulo `tables`) en el navegador.

## Escenario a validar (mapeado a los Acceptance Scenarios del spec)

1. **Crear y cobrar por adelantado un pedido de mostrador**: seleccionar una mesa libre, crear un
   pedido de mostrador con al menos un producto, y cobrarlo con "Cobrar, Facturar y Enviar a
   Cocina" (`checkout_and_send`) — esto deja el pedido en estado `'pagada'` mientras cocina todavía
   no lo prepara.
2. **Confirmar el estado correcto mientras cocina trabaja** (ya funciona hoy, sin cambios):
   verificar que el panel central muestra el pedido y el panel derecho ofrece "Cuenta de la mesa"
   con Total/"Imprimir Factura"/"Liberar Mesa".
3. **Marcar el pedido como listo**: pulsar "Marcar pedido listo" en el panel central.
4. **Verificar que el pedido sigue visible** (el bug corregido): confirmar que el panel central
   **no** cae al mensaje de "mesa libre" — sigue mostrando el pedido. Confirmar que el panel
   derecho **sigue** ofreciendo "Cuenta de la mesa" (Total/"Imprimir Factura"/"Liberar Mesa"), no
   "Pedido de mostrador"/"+ Crear pedido nuevo".
5. **Verificar el badge de la tarjeta**: en la grilla de mesas, confirmar que la tarjeta de esa
   mesa muestra el estado "Listo" (no "Ocupada" genérico).
6. **Liberar la mesa**: pulsar "Liberar Mesa" y confirmar que, recién en ese momento, el pedido deja
   de mostrarse y la mesa vuelve a verse "Libre" — comportamiento ya implementado, sin cambios.
7. **Caso de dos pedidos en la misma mesa**: repetir el escenario con una segunda orden en la misma
   mesa que siga en preparación, y confirmar que ambos pedidos siguen contando como consumo de la
   mesa — ninguno desaparece antes de tiempo.

## Verificación automatizada

```bash
cd ../pos-heladeria
pnpm test -- pos-terminal.store   # existente + los dos casos nuevos de research.md §4
```

(Ajustar el runner exacto —`pnpm test`, `ng test`, `npx vitest`— al script real configurado en
`package.json` del proyecto en el momento de implementar; hoy es `ng test`.)
