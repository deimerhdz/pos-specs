# Quickstart: Validar el rediseño responsive de la Terminal de Mesas

Guía de validación manual/end-to-end de las 5 historias de usuario de `spec.md`. No sustituye los
tests automatizados de `tasks.md` — sirve para confirmar el comportamiento observable descrito en
los Acceptance Scenarios.

## Prerrequisitos

- `pos-backend` corriendo localmente con al menos un tenant de prueba con mesas configuradas
  (ver `specs/000-reconocimiento/` para el flujo de entorno ya documentado).
- `pos-heladeria` corriendo (`ng serve`) apuntando a ese backend.
- Un usuario de staff con rol Cajero (acceso completo) y, opcionalmente, uno con rol Mesero (spec
  075) para validar FR-022 con ambos roles.
- Herramientas de dev del navegador para simular anchos de viewport (DevTools → responsive mode) en
  al menos tres anchos: ~1280px (escritorio), ~900px (tablet), ~390px (móvil).

## Historia 1 — Cuadrícula responsive

1. Abrir la Terminal de Mesas con el viewport en ~1280px.
2. **Verificar**: la grilla muestra 4 tarjetas por fila, sin flechas de desplazamiento horizontal;
   si hay más de 4 mesas, aparecen en filas adicionales con scroll vertical (FR-001, FR-002).
3. Reducir el viewport a ~900px. **Verificar**: 3 tarjetas por fila (FR-003).
4. Reducir el viewport a ~390px. **Verificar**: 2 tarjetas por fila (FR-004), y el texto de cada
   tarjeta aparece abreviado (por ejemplo "4 prod · 14m", "En prep.") (FR-007).
5. Abrir la pestaña "Para llevar" (o "Domicilios") con pedidos pendientes de cobro creados desde la
   creación manual (spec 059). **Verificar**: usan la misma cuadrícula responsive (FR-005).
6. Seleccionar cualquier tarjeta en cualquiera de los tres anchos. **Verificar**: mismo
   comportamiento de selección ya existente (resalta la tarjeta, muestra el detalle).

## Historia 2 — Contadores de ocupación

1. Con la pestaña "Mesas" activa, anotar manualmente cuántas tarjetas hay de cada estado.
2. **Verificar**: el resumen superior (total/ocupadas/libres/pendientes) coincide exactamente con
   el conteo manual (SC-002).
3. **Verificar**: cada filtro ("Todas"/"Libres"/"Ocupadas"/"Pendientes") muestra el mismo número que
   le corresponde junto a su etiqueta, y que ese número coincide con el del resumen superior
   (FR-011).
4. Cobrar/liberar una mesa ocupada. **Verificar**: ambos contadores se actualizan solos, sin
   recargar la página (FR-012).

## Historia 3 — Barra superior operativa

1. Sin ningún turno de caja abierto en la caja registradora de la sesión: abrir la Terminal de
   Mesas. **Verificar**: aparece el botón de abrir turno con el atajo `F1` visible junto a él
   (FR-015).
2. Presionar `F1` (o pulsar el botón). **Verificar**: se dispara el mismo flujo de apertura de
   turno ya usado en el módulo de Caja (mismo formulario/validaciones), sin duplicar lógica
   (FR-016).
3. Completar la apertura del turno. **Verificar**: la barra superior ahora muestra el nombre del
   cajero de turno en vez del botón de abrir turno (FR-017).
4. **Verificar**: la barra muestra un reloj con hora y fecha actuales (FR-013), un indicador de
   sincronización (FR-014), y una etiqueta fija "POS" sin acción al pulsarla (FR-018) — sin ningún
   ícono de candado (FR-019).
5. Desconectar la red del cliente (DevTools → Network → Offline). **Verificar**: el indicador de
   sincronización cambia a "sin conexión" sin bloquear la pantalla ni perder la selección actual;
   reconectar y verificar que vuelve a "en línea" solo.

## Historia 4 — "Atendido por"

1. Con un usuario Cajero autenticado, crear un pedido de mesa (mostrador o desde la Terminal de
   Mesas). **Verificar**: la tarjeta de esa mesa muestra "Atendido por" con el nombre de ese
   cajero (FR-022, SC-004).
2. Repetir con un usuario Mesero (spec 075). **Verificar**: se muestra igual, sin distinción de rol
   (FR-022, decisión de Clarifications).
3. Abrir el menú QR como cliente y crear un pedido sin staff. **Verificar**: su tarjeta NO muestra
   ninguna referencia de "Atendido por" (FR-023).
4. Revisar una mesa libre sin pedido. **Verificar**: tampoco muestra "Atendido por" (FR-023).

## Historia 5 — Panel sin selección

1. Abrir la Terminal de Mesas sin seleccionar ninguna mesa/pedido. **Verificar**: el panel derecho
   muestra el mensaje de bienvenida (FR-025), la lista de atajos de teclado (buscar mesa, crear
   pedido nuevo, abrir turno de caja, con sus teclas — FR-026), y el botón fijo "+ Crear pedido
   nuevo" en la parte inferior (FR-027).
2. Seleccionar cualquier mesa u pedido. **Verificar**: el mensaje de bienvenida y la lista de
   atajos desaparecen, reemplazados por el detalle/cobro ya existente, sin cambios de contenido
   (FR-028).

## Regresión rápida (no negociable)

- Ningún dato hoy visible en una tarjeta (estado, productos, tiempo, total, badge de "N pedidos")
  desaparece en ningún ancho (FR-006, SC-005).
- Los colores de cada estado y su texto siguen siendo exactamente los mismos que hoy — ninguno
  cambia (FR-029, FR-030): "Libre", "Ocupada", "En preparación", "Listo", "Pago pendiente", "Por
  confirmar", "Reservada".
- Los atajos F2 (buscar), F3 (crear pedido), ESC (cancelar) y Ctrl/Cmd+P (imprimir, en el diálogo de
  éxito) siguen funcionando exactamente igual que antes de este rediseño.
