# Contrato: cuenta de la mesa, división y cobro combinado — sin cambios de API

Cubre FR-006 a FR-009. **Ningún endpoint de este contrato cambia** — la spec expone en la interfaz
capacidad que el backend ya ofrece completa.

## `GET /table-sessions/{table_session_id}/bill`

- Sin cambios. Sigue siendo la fuente del detalle de cuenta (ítems, cantidades, subtotal,
  descuentos/promociones aplicados, total) que `SessionBillPanelComponent` debe mostrar sin salir
  de la Terminal de Mesas (FR-006) — ya lo hace hoy; la fase de tareas verifica que ningún dato de
  esta respuesta falte en el panel.

## `POST /table-sessions/{table_session_id}/close`

- Sin cambios. Acepta `billing_mode: "unified" | "split"`:
  - `unified`: un único `payments[]` combinando métodos (efectivo + cualquier otro) — reutilizado
    tal cual por FR-008.
  - `split`: N bloques, cada uno con su propia asignación de ítems/unidades a un comensal
    (`AssignRow.units[]`, nunca porcentual, spec 010 FR-005) y su propio `payments[]` — reutilizado
    tal cual por FR-007/FR-008.
- El cambio (`change_given`) solo puede originarse de un excedente pagado en efectivo (spec 010,
  FR-020) — regla ya vigente, sin modificación.
- La factura se emite dentro de la misma transacción de cierre vía `build_sale` (spec 011) — ya
  satisface FR-009 (factura automática, sin pasos adicionales) sin cambios.

## Lo que sí cambia (fuera de este contrato)

Únicamente la interfaz (`SessionBillPanelComponent`/`SplitBillPanelComponent`): tamaño de texto y
de controles (FR-010/FR-011) y, donde aplique, etiquetas de estado no dependientes del color
(FR-003) — ningún payload ni endpoint nuevo.
