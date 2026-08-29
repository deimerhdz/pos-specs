# Quickstart: validación manual de la corrección de "Liberar Mesa"

## Prerrequisitos

- `pos-backend` corriendo localmente con una base de datos de prueba.
- `pos-heladeria` corriendo localmente (`ng serve`) apuntando a ese backend.
- Un turno de caja abierto y al menos una mesa libre.

## Escenario 1 — Liberar una mesa con un pedido cancelado (User Story 1)

1. En la Terminal de Mesas, crear un pedido manual sobre una mesa libre (botón "+ Crear pedido
   nuevo" / F3), agregar al menos un producto y guardarlo — el ítem queda `'pendiente'` en cocina.
2. Sin marcarlo como listo, cancelarlo ("Rechazar pedido").
3. Pulsar "Liberar Mesa".

**Resultado esperado (antes del fix)**: error "Hay ítems sin terminar en cocina; anúlalos o espera
a que estén listos." — la mesa queda bloqueada.

**Resultado esperado (después del fix)**: la mesa se libera sin error y vuelve a aparecer como
"Libre" en la grilla.

## Escenario 2 — Un pedido `'pagada'` con cocina en curso sigue bloqueando (regresión protegida)

1. Crear un pedido de mostrador cobrado por adelantado (`hold_for_payment`) sobre una mesa, sin
   dejar que cocina lo termine.
2. Pulsar "Liberar Mesa".

**Resultado esperado (sin cambios, antes y después)**: sigue rechazando con el mismo error — este
caso no debe cambiar (Principio II, ver spec.md Acceptance Scenario 2).

## Escenario 3 — Avisos repetidos no se apilan (User Story 2)

1. Provocar dos veces seguidas, sin resolver la causa entre intentos, cualquier error visible con un
   aviso (por ejemplo, repetir el Escenario 1 dos veces sin aplicar el fix, o cualquier otro error
   reproducible dos veces con el mismo texto).
2. Contar cuántas tarjetas de aviso quedan visibles.

**Resultado esperado**: una sola tarjeta, no una copia por cada intento. Provocar un error de texto
distinto en el medio debe seguir mostrando una tarjeta nueva independiente (no debe ocultarse por
la deduplicación).

## Verificación automatizada equivalente

- Backend: `pytest app/characterization_tests/test_table_sessions_service.py -q` — nuevo test junto
  a `test_release_paid_session_409_con_cocina_en_curso_sobre_pedido_pagado` (línea 647) que cubre
  exactamente el Escenario 1, más ese mismo test existente sin cambios cubriendo el Escenario 2.
- Frontend: `ng test` sobre un `toast.service.spec.ts` nuevo que cubre el Escenario 3
  (ver contracts/api-contract.md, tabla de `ToastService`).
