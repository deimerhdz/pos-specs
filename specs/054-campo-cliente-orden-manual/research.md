# Research: Campo "Cliente" con valor por defecto "Consumidor final" en la creación de orden manual

Todas las incógnitas de esta spec son de diseño técnico dentro del frontend `pos-heladeria`
(Angular, módulo `tables`, componente `manual-order-page.component.ts`); no hay cambios de backend
que investigar — confirmado leyendo `pos-backend` (spec.md, sección de alcance): `customer_name` ya
existe en `CustomerOrder`, en `OrderCreate` y en el endpoint `POST /orders`, y
`createManualOrderFromDraft()` ya lo envía.

## D1 — Dónde aplicar el valor por defecto "Consumidor final"

**Decision**: Aplicar el valor por defecto **solo dentro de `ManualOrderPageComponent`**, no en el
método compartido `PosTerminalStore.selectTable()` (`pos-terminal.store.ts:1020-1038`). Se agrega
un método privado `applyDefaultCustomerName()` en el componente que hace
`if (!this.store.customerName().trim()) this.store.customerName.set('Consumidor final')`, llamado:
(a) al final de `ngOnInit()`, después de la llamada a `store.selectTable(tableId)` cuando hay
parámetro de ruta; (b) dentro del método `selectTable(id)` del componente (el que usa
`(ngModelChange)` del select de mesas de spec 053), justo después de `this.store.selectTable(id)`.

**Rationale**: `selectTable()` (`pos-terminal.store.ts:1020-1038`) es compartido por esta pantalla y
por `table-sessions.component.ts` (Terminal de Mesas, otra instancia del mismo store). Su rama sin
pedido existente deja `customerName` en `''` **a propósito**, según su propio comentario
(línea 1042-1044): evitar que un relleno tipo "Cliente Mesa 3" quede en el documento fiscal. Ese
comentario sigue siendo válido para la Terminal de Mesas (donde el campo "Cliente" es de solo
lectura desde spec 049 y nunca se edita ahí). Cambiar `selectTable()` en sí mismo aplicaría
"Consumidor final" también en la Terminal de Mesas sin que esta spec lo haya pedido ni analizado —
violaría Principio V/VI (mezclar un cambio no solicitado en otra pantalla). Aplicar el default
**después**, solo en el componente de esta spec, dejar
`selectTable()` intacto y limita el cambio exactamente a la pantalla que spec.md describe.

**Alternatives considered**:
- *Cambiar el valor de `''` a `'Consumidor final'` directamente en `selectTable()`*: rechazado por
  las razones de arriba — cambiaría comportamiento de la Terminal de Mesas sin necesidad ni
  autorización de esta spec.
- *Agregar un parámetro opcional a `selectTable()` (p. ej. `selectTable(id, { defaultCustomerName:
  'Consumidor final' })`)*: rechazado por sobre-ingeniería — un `if` de dos líneas en el componente
  ya resuelve el requisito sin tocar la firma de un método compartido por otra pantalla.

## D2 — Interacción "readonly con opción de editar"

**Decision**: Un `<input>` con `[readOnly]="!editandoCliente()"` (signal local del componente,
inicial `false`), envuelto en un `<div class="relative">`, con un botón `✏️` posicionado
`absolute right-2` que llama a `toggleEditarCliente()` (pone `editandoCliente` en `true` y enfoca
el input). El input pierde el modo edición en `(blur)`, llamando a `onClienteBlur()` (pone
`editandoCliente` en `false` y aplica el valor por defecto si quedó vacío, research.md D3).

**Rationale**: es el mismo patrón estructural (`relative` + botón `absolute` con ícono, alternando
un estado del input) que ya usa `password-input.component.ts:26-53` para su toggle mostrar/ocultar
contraseña — la diferencia es que aquí alterna `readOnly` en vez de `type`. El ícono `✏️` (en vez de
un ícono SVG de `app-icon`, que no tiene ningún caso "edit"/lápiz hoy) ya es el mismo que usa el
botón "Editar" de `tables-page.component.ts:119-123` — se reutiliza el mismo emoji en vez de
extender el set de íconos compartido para un solo uso.

**Alternatives considered**:
- *Extender `IconComponent` con un ícono "edit" nuevo*: rechazado — hay precedente directo en el
  propio proyecto de usar el emoji `✏️` para exactamente esta acción ("Editar"); agregar un ícono
  SVG nuevo sería una extensión de un componente compartido sin necesidad, para un único
  consumidor.
- *Crear un componente reutilizable `EditableInputComponent`*: rechazado por alcance — esta es la
  única pantalla con este requisito hoy; construir una abstracción reutilizable sin un segundo
  consumidor real viola Principio V (no generalizar antes de que haga falta).

## D3 — Nunca guardar un nombre de cliente vacío (FR-005, Edge Cases)

**Decision**: Dos puntos de defensa, no uno solo:
1. `onClienteBlur()` (D2): si `store.customerName().trim()` queda vacío al perder el foco, se
   restaura `'Consumidor final'`.
2. `confirm()` (método existente del componente, `manual-order-page.component.ts:252-257`): antes
   de llamar a `store.createManualOrderFromDraft()`, se agrega la misma comprobación
   (`applyDefaultCustomerName()`) — cubre el caso borde en que el mesero confirma la orden mientras
   el campo sigue en modo edición y vacío, sin haber perdido el foco todavía (Edge Cases, spec.md).

**Rationale**: `(blur)` no se dispara si el mesero nunca sale del campo antes de hacer clic en
"Confirmar y Enviar" (p. ej. arrastra el mouse directo al botón) — depender solo del blur dejaría
un hueco exactamente en el escenario que spec.md marca como edge case explícito. Repetir la misma
comprobación barata (una función idempotente) en ambos puntos es más simple que inventar un tercer
mecanismo (p. ej. validar en el propio `createManualOrderFromDraft()` del store, que serviría a
todas las pantallas y no solo a esta, otra vez mezclando alcance con la Terminal de Mesas).

**Alternatives considered**: validar/normalizar dentro de `createManualOrderFromDraft()` (el store,
compartido) — rechazado, mismo argumento de D1: esa función también la usa la Terminal de Mesas
(vía otro flujo), y agregar ahí un default de "Consumidor final" excede lo que esta spec pidió.

## Resumen de impacto en tests existentes

`manual-order-page.component.spec.ts` (post-053, 21 casos): ninguno de los casos existentes
depende del contenido exacto del bloque "Tipo de Orden"/"Mesas" más allá de sus propios textos ya
cubiertos (spec 051-053) — agregar un campo "Cliente" nuevo entre "Mesas" y "Nueva orden" no rompe
ningún `querySelector`/`textContent` existente (ninguno usa `nextElementSibling` sobre ese bloque
salvo el ya retirado en spec 053). Los 4 casos que llaman a `createManualOrderFromDraft()`
(directamente o vía "Confirmar y Enviar") no aseveran nada sobre `customer_name` hoy — no necesitan
cambiar, pero se agrega cobertura nueva que sí lo haga (Fase de implementación). 0 tests
`"CONGELA comportamiento actual:"` en `pos-heladeria/src/` (mismo hallazgo que specs 045-053).
