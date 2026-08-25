---

description: "Task list template for feature implementation"
---

# Tasks: Estado "Pagada" Correcto y Formato de Moneda Reutilizable

**Input**: Documentos de diseño de `/specs/035-estado-pagado-formato-moneda/`

**Prerequisitos**: plan.md, spec.md, research.md, data-model.md, contracts/, quickstart.md

**Tests**: no se solicitaron tests dedicados como una fase aparte, pero Historia 1 exige
verificar explícitamente que los characterization tests protegidos no se rompen (Principio
III) y agregar los casos nuevos que la propia spec introduce — esas tareas sí están incluidas
dentro de la fase de Historia 1.

**Organización**: las tareas se agrupan por historia de usuario (spec 035, Historias 1-2),
verificable y desplegable cada una por separado (Principio VI).

## Formato: `[ID] [P?] [Story] Descripción`

- **[P]**: puede ejecutarse en paralelo (archivo distinto, sin dependencias pendientes)
- **[Story]**: historia de usuario a la que pertenece (US1, US2)
- Cada tarea incluye la ruta de archivo exacta

## Convenciones de ruta

- Backend: `/home/deimer/Documents/projects/inventario/pos-backend/app/api/v1/orders/`
- Frontend: `/home/deimer/Documents/projects/inventario/pos-heladeria/src/app/`
- Registro de decisión de negocio: `/home/deimer/Documents/projects/inventario/pos-specs/specs/000-reconocimiento/registro-de-anomalias.md`

---

## Fase 1: Setup

Sin tareas de setup — ambas historias reutilizan la aplicación ya en producción (dos repos
existentes), sin scaffolding, dependencia, ni infraestructura nueva (Technical Context de
`plan.md`).

## Fase 2: Foundational

Sin tareas fundacionales compartidas — Historia 1 (backend, `pos-backend`) e Historia 2
(frontend, `pos-heladeria`) no comparten ningún archivo, modelo, ni bloqueo entre sí. Cada una
puede empezar de inmediato.

---

## Fase 3: Historia de Usuario 1 - El estado de una orden cobrada en Terminal de Mesas queda "pagada" de verdad (Priority: P1)

**Objetivo**: que `checkout_and_send` fije `status = 'pagada'` en la misma transacción en que
registra la venta, sin marcar prematuramente como pagada ninguna orden QR sin venta, y sin
perder la protección contra liberar/mover/fusionar una mesa con comida todavía en
preparación.

**Independent Test**: cobrar una comanda desde Terminal de Mesas y consultar el pedido
directamente — su `status` debe ser `'pagada'`; con ítems todavía sin `listo`, intentar
liberar/mover/fusionar su mesa debe seguir respondiendo 409; una vez todos los ítems
`listo`/`anulado`, esas mismas operaciones deben permitirse.

### Implementación de la Historia 1

- [ ] T001 [US1] Registrar la decisión de negocio en
  `specs/000-reconocimiento/registro-de-anomalias.md`: qué cambia (`checkout_and_send` pasa a
  fijar `status = 'pagada'`, revirtiendo para este camino puntual la decisión de spec 029),
  por qué, quién y cuándo lo autoriza, y qué funcionalidades quedan afectadas (las tres
  funciones de `tables_advanced.py` de T004-T006). Bloqueante por Principio II — ningún
  cambio de código de esta historia se implementa antes de esta entrada.
- [ ] T002 [P] [US1] En
  `pos-backend/app/api/v1/orders/checkout.py`, función `checkout_and_send` (líneas 421-505),
  agregar `order.status = "pagada"` inmediatamente después de la llamada a
  `_deduct_and_open(db, order, cashier)` (línea 490) — en la misma transacción, antes del
  `db.commit()`. No se modifica `_deduct_and_open` en sí (research.md, Decisión 1). (depende
  de T001)
- [ ] T003 [P] [US1] En `pos-backend/app/api/v1/orders/tables_advanced.py`, agregar una
  función privada nueva (ej. `_order_blocks_table(db, order) -> bool` para uso en Python
  sobre una orden ya cargada, o su expresión SQL equivalente) que implemente el predicado de
  `research.md` Decisión 2: `status != 'cancelada' AND (status != 'pagada' OR existe algún
  OrderItem de la orden con estado_cocina NOT IN ('listo', 'anulado'))`. (depende de T001)
- [ ] T004 [US1] En `tables_advanced.py`, actualizar `_active_orders_on_table` (líneas 23-29)
  para usar la condición de T003 en vez de `status.notin_(TERMINAL)` — agregar el `EXISTS`
  correlacionado contra `order_items` para el caso `status = 'pagada'`. (depende de T003)
- [ ] T005 [US1] En `tables_advanced.py`, actualizar el chequeo de la orden que se mueve en
  `move_order` (línea 49, hoy `if order.status in TERMINAL`) para usar el predicado de T003
  sobre la orden ya cargada (con sus `items`). (depende de T003)
- [ ] T006 [US1] En `tables_advanced.py`, actualizar el chequeo de las órdenes a fusionar en
  `merge_orders` (línea 83, hoy `if any(o.status in TERMINAL for o in orders)`) para usar el
  predicado de T003 sobre cada orden ya cargada. `group_bill` (líneas 94-125) **no** se toca
  — su exclusión de `pagada`/`cancelada` del total ya es correcta y no depende de esta spec.
  (depende de T003)
- [ ] T007 [P] [US1] Actualizar la aserción de
  `app/characterization_tests/test_orders_checkout.py::test_checkout_and_send_cobra_descuenta_y_abre_a_cocina`
  (línea 566) de `self.assertEqual(order.status, "abierta")` a
  `self.assertEqual(order.status, "pagada")`. No es un test `"CONGELA comportamiento
  actual:"` (su docstring dice "Comportamiento nuevo, spec 028") — actualizarlo no requiere
  el procedimiento del Principio III, solo reflejar el comportamiento que esta spec autoriza.
  (depende de T002)
- [ ] T008 [P] [US1] Agregar un test nuevo en
  `app/characterization_tests/test_orders_tables_advanced.py`: una orden `'pagada'` (creada
  vía `checkout_and_send` o fijada directamente) con al menos un `OrderItem` en
  `estado_cocina='pendiente'` → `set_table_status(..., 'libre')`, `move_order`, y
  `merge_orders` siguen respondiendo `409` (Acceptance Scenario 3 de Historia 1). (depende de
  T004, T005, T006)
- [ ] T009 [US1] Agregar un test nuevo en el mismo archivo: la misma orden `'pagada'` con
  **todos** sus ítems en `estado_cocina IN ('listo', 'anulado')` → las tres operaciones ahora
  se permiten (Escenario 2, paso 3 de `quickstart.md`). Mismo archivo que T008 — secuencial.
  (depende de T004, T005, T006)
- [ ] T010 [US1] Correr
  `python -m unittest app.characterization_tests.test_orders_tables_advanced app.characterization_tests.test_orders_checkout -v`
  y confirmar que **ningún** test `"CONGELA comportamiento actual:"` requirió modificación
  (research.md, Decisión 2, ya verificó por lectura que
  `test_set_table_status_409_con_ordenes_activas_y_ok_sin_ellas` usa una orden sin ítems y
  sigue pasando tal cual). Verificación, no código. (depende de T007, T008, T009)

**Checkpoint**: cobrar desde Terminal de Mesas deja `status = 'pagada'` en base de datos sin
perder la protección de mesa mientras cocina trabaja — Historia 1 verificable de forma
independiente.

---

## Fase 4: Historia de Usuario 2 - Cualquier campo de precio o monto se auto-formatea como moneda colombiana (Priority: P2)

**Objetivo**: un único control de entrada reutilizable, con formato de miles en vivo (punto,
igual que `shared/money.ts`), usado en los ~12 campos de precio/monto ya identificados.

**Independent Test**: escribir un número en cualquiera de los campos migrados y verificar que
se ve con separador de miles mientras se escribe, y que el valor guardado/enviado sigue
siendo el número correcto sin el separador.

### Implementación de la Historia 2

- [ ] T011 [US2] Crear
  `pos-heladeria/src/app/shared/money-input/money-input.component.ts`: componente standalone,
  `ControlValueAccessor` auto-inyectando `NgControl` en el constructor (mismo patrón que
  `shared/password-input/password-input.component.ts` y
  `shared/searchable-select/searchable-select.component.ts`), `@Input() decimals = 0`,
  formateo con `Intl.NumberFormat('es-CO', { maximumFractionDigits: decimals })` en cada
  evento `input` preservando la posición del cursor, `writeValue`/`onChange` siempre con un
  `number | null` limpio (nunca la cadena formateada), `null`/`undefined` deja el campo
  vacío sin forzarlo a `0` (FR-009). Ver `contracts/money-input-component.md` para el
  contrato completo.
- [ ] T012 [P] [US2] Migrar "Fondo inicial (base de efectivo)" en
  `pos-heladeria/src/app/modules/cash-register/components/cash-open.component.ts:76-87` de
  `<input type="number">` a `<app-money-input>`. (depende de T011)
- [ ] T013 [P] [US2] Migrar el campo "Valor" en
  `pos-heladeria/src/app/modules/cash-register/components/cash-movement-modal.component.ts:39-49`
  a `<app-money-input>`. (depende de T011)
- [ ] T014 [P] [US2] Migrar "Efectivo contado" (arqueo parcial) en
  `pos-heladeria/src/app/modules/cash-register/components/cash-dashboard.component.ts:172` a
  `<app-money-input>`. (depende de T011)
- [ ] T015 [P] [US2] Migrar el precio por presentación/variante en
  `pos-heladeria/src/app/modules/products/pages/product-form.component.ts:230-232` a
  `<app-money-input>`. (depende de T011)
- [ ] T016 [P] [US2] Migrar "Precio extra" en
  `pos-heladeria/src/app/modules/option-groups/components/option-form.component.ts:47-49` a
  `<app-money-input [decimals]="2">` (hoy admite centavos, `step="0.01"`). (depende de T011)
- [ ] T017 [P] [US2] Migrar "Costo unitario" en
  `pos-heladeria/src/app/modules/inventory/components/inventory-item-form.component.ts:82-84`
  a `<app-money-input [decimals]="2">`. (depende de T011)
- [ ] T018 [P] [US2] Migrar "Costo unit." por línea de compra en
  `pos-heladeria/src/app/modules/inventory/components/purchase-form.component.ts:84-86` a
  `<app-money-input [decimals]="2">`. (depende de T011)
- [ ] T019 [P] [US2] Migrar en
  `pos-heladeria/src/app/modules/promotions/pages/promotions-page.component.ts`: "Precio del
  combo" (líneas 737 y 983) siempre a `<app-money-input>`; el campo dual porcentaje/monto
  (líneas 463 y 757) a `<app-money-input>` únicamente cuando `form.type !== 'percent'` (FR-011
  — cuando es porcentaje, se conserva el `<input type="number">` simple ya existente).
  (depende de T011)
- [ ] T020 [P] [US2] Migrar el precio de paquete en
  `pos-heladeria/src/app/modules/promotions/components/scope-picker.component.ts:305-314` a
  `<app-money-input>`, preservando que dejarlo vacío siga significando "hereda el valor por
  defecto del paquete" (FR-009 — no forzar a `0`). (depende de T011)
- [ ] T021 [P] [US2] Migrar el monto que paga el comensal en
  `pos-heladeria/src/app/modules/tables/components/payment-input.component.ts:51-58,85-92` a
  `<app-money-input>`. (depende de T011)
- [ ] T022 [P] [US2] Migrar "Monto recibido" en
  `pos-heladeria/src/app/modules/tables/components/payment-attempt-review-panel.component.ts:46-53`
  a `<app-money-input>`. (depende de T011)
- [ ] T023 [P] [US2] Migrar "Precio mensual"/"Precio anual" en
  `pos-heladeria/src/app/modules/super-admin/components/plan-form.component.ts:107-129` a
  `<app-money-input [decimals]="2">` (dos campos). (depende de T011)

**Checkpoint**: los ~12 campos de precio/monto identificados muestran formato de miles en
vivo con el mismo valor limpio de siempre por debajo — Historia 2 verificable de forma
independiente.

---

## Fase Final: Pulido y verificación cruzada

**Propósito**: validar el conjunto y confirmar que no hay regresiones fuera del alcance de
esta spec.

- [ ] T024 [P] Correr `cd pos-heladeria && npx ng build` y confirmar que compila limpio tras
  la migración completa de Historia 2 (sin referencias sueltas al `<input type="number">`
  original en los 9 archivos migrados).
- [ ] T025 [P] Correr la suite completa de characterization tests de `orders`/`table_sessions`
  en `pos-backend` (no solo los dos archivos de T010) y confirmar que no hay regresiones
  fuera de las ya esperadas en T007-T009.
- [ ] T026 Ejecutar los 4 escenarios de `quickstart.md` completos contra un entorno local
  (`pos-backend` + `pos-heladeria` corriendo).

---

## Dependencias y orden de ejecución

### Dependencias entre fases

- **Setup/Foundational**: sin tareas — ambas historias empiezan de inmediato.
- **Historia 1 (Fase 3)** y **Historia 2 (Fase 4)** son completamente independientes entre sí
  (repos, archivos y equipos distintos) — pueden implementarse en paralelo o en cualquier
  orden.
- **Pulido (Fase Final)**: depende de que las historias que se vayan a entregar estén
  completas (T024 depende de Historia 2; T025/T026 dependen de ambas si se entregan juntas).

### Dentro de la Historia 1

- T001 (registro de decisión) antes de T002/T003 (Principio II).
- T003 (predicado nuevo) antes de T004/T005/T006 (los tres consumidores, mismo archivo
  `tables_advanced.py` — secuenciales entre sí).
- T002 antes de T007 (la aserción del test depende del comportamiento real).
- T004-T006 antes de T008/T009 (los tests nuevos ejercitan las tres funciones ya
  actualizadas); T008 antes de T009 (mismo archivo).
- T007, T008, T009 antes de T010 (verificación final de no-regresión).

### Dentro de la Historia 2

- T011 (el componente) antes de cualquiera de T012-T023 (las migraciones).
- T012-T023 son completamente independientes entre sí (archivos distintos, ningún campo
  depende de otro) — todas ejecutables en paralelo una vez completada T011.

### Oportunidades de paralelismo

- T002 y T003 pueden avanzar en paralelo (archivos `checkout.py` vs `tables_advanced.py`)
  una vez completada T001.
- T007 y T008 pueden avanzar en paralelo (archivos de test distintos) una vez cumplidas sus
  respectivas dependencias.
- Las doce migraciones de Historia 2 (T012-T023) pueden lanzarse todas juntas una vez lista
  T011 — es el bloque de paralelismo más grande de esta spec.
- T024 y T025 pueden ejecutarse en paralelo entre sí al final.

---

## Ejemplo de ejecución en paralelo: Historia 2

```bash
# Una vez completado T011 (MoneyInputComponent), lanzar juntas (ejemplos, hay 12 en total):
Task: "Migrar Fondo inicial en cash-open.component.ts a <app-money-input>"
Task: "Migrar costo unitario en inventory-item-form.component.ts a <app-money-input [decimals]=2>"
Task: "Migrar precio del combo y descuento dual en promotions-page.component.ts"
Task: "Migrar monto que paga el comensal en payment-input.component.ts"
```

---

## Estrategia de implementación

### MVP (Historia 1 sola)

Historia 1 es la corrección de datos que originó el reporte — tiene valor por sí sola sin
Historia 2.

1. Completar Fase 3: Historia 1 (T001-T010).
2. **DETENER Y VALIDAR**: ejecutar los Escenarios 1-3 de `quickstart.md`.
3. Desplegar/demostrar el MVP si está listo.

### Entrega incremental

1. Historia 1 (estado "pagada" correcto) → validar (Escenarios 1-3) → desplegar.
2. Historia 2 (input de moneda reutilizable) → validar (Escenario 4) → desplegar.
3. Fase Final (pulido y no-regresión) → cierre de la funcionalidad.

Al ser completamente independientes, también pueden desarrollarse **en paralelo** por dos
personas/equipos distintos desde el primer día.

---

## Notas

- [P] = archivos distintos, sin dependencias pendientes.
- Historia 1 e Historia 2 no comparten ningún archivo ni comportamiento — ver Principio VI en
  `plan.md`.
- T001 es un bloqueante de proceso (Principio II), no de código — sin él, T002/T003 no deben
  comenzar aunque técnicamente nada lo impida a nivel de archivos.
- Ningún test `"CONGELA comportamiento actual:"` se modifica en esta spec (research.md,
  Decisión 2) — T007 actualiza un test que explícitamente no lleva esa marca.
