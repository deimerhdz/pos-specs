# Contrato: `POST /orders/{order_id}/confirm` se mantiene sin cambios

- **Request/Response**: sin ningún cambio de forma ni de precondición — sigue exigiendo un Intento
  de Pago `confirmado` (spec 024, FR-017) y sigue devolviendo la `CustomerOrder` actualizada.
- **Efecto**: sin cambios de comportamiento propio — sigue descontando inventario y avanzando
  `recibida → abierta` exactamente igual que hoy.
- **Lo único que cambia es quién lo llama**: en el flujo normal de la Terminal de Mesas, ya no
  existe ningún botón que lo invoque manualmente (research.md, Decisión 3) — para cuando un
  Intento de Pago queda `confirmado`, la orden ya fue enviada a cocina como parte de esa misma
  operación (`payment-confirmation.md`). Este endpoint queda disponible como vía de recuperación
  técnica, no como parte del flujo de uso diario.
- **Consumidores fuera de la Terminal de Mesas** (si existieran, p. ej. herramientas internas o
  scripts de soporte) no requieren ningún cambio — su contrato es idéntico al de antes de esta
  spec.
