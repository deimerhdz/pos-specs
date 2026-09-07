# Quickstart: Validar las correcciones responsive de la Terminal de Mesas

Guía de validación manual/end-to-end de las seis historias de `spec.md`. No sustituye los tests
automatizados de `tasks.md` — confirma el comportamiento observable de los Acceptance Scenarios y
los Success Criteria.

## Prerrequisitos

- `pos-backend` corriendo localmente con un tenant de prueba: mesas configuradas, al menos una
  promoción vigente, y método de pago en efectivo.
- `pos-heladeria` corriendo (`ng serve`) apuntando a ese backend.
- Un usuario de staff con rol Cajero (o Admin).
- DevTools del navegador en modo responsive, con tres anchos a mano:
  **~1280px (escritorio)**, **~900px (tablet)**, **~390px (móvil)**.
- Datos sembrados:
  - Un pedido de **Domicilio** pendiente de cobro con subtotal de productos ≈ $25.000 y
    `delivery_fee` = $6.000, sin descuento.
  - Un pedido de **Domicilio** con `delivery_fee` = 0 (envío gratis).
  - Un pedido de **Domicilio** con una promoción aplicada (descuento > 0).
  - Un pedido (mesa o para llevar) con **≥ 6 productos** y con una **dirección de entrega larga**
    (si es de Domicilio) — para las Historias 3, 4 y 5.
  - Un pedido "Para llevar" pendiente de cobro.

## Comandos

```bash
# En pos-heladeria/
ng serve
ng test --watch=false          # suite completa
ng test --watch=false --include='**/pos-terminal.store.spec.ts'
ng test --watch=false --include='**/table-sessions.component.spec.ts'
ng test --watch=false --include='**/dashboard-layout.component.spec.ts'
```

Antes de empezar con la Historia 6: confirmar que la entrada `A-71` ya está en
`specs/000-reconocimiento/registro-de-anomalias.md` (Constitution Check, Principio II — ver
[contracts/ui-terminal-mesas-fixes.md](./contracts/ui-terminal-mesas-fixes.md)).

---

## Historia 1 — Total de la tarjeta de un pedido de Domicilio (P1)

1. Abrir la Terminal de Mesas → pestaña "Domicilios".
2. En la tarjeta del pedido con subtotal $25.000 + domicilio $6.000: **verificar** que el total
   mostrado es **$31.000**, no $25.000 (FR-001, SC-001).
3. Seleccionar ese pedido → mirar el total del panel de detalle/cobro. **Verificar** que es
   **idéntico** al de la tarjeta ($31.000) (FR-003, escenario 2).
4. En la tarjeta del pedido con promoción: **verificar** que el total sigue la fórmula
   `subtotal − descuento + domicilio` (el mismo número que aparece al seleccionarlo y abrir el
   cobro), no una fórmula propia de la tarjeta (FR-002).
5. En la tarjeta del pedido con `delivery_fee` = 0: **verificar** que el total = subtotal de
   productos, sin error ni descuadre (FR-006, Edge Case "domicilio cero").
6. Abrir la pestaña "Para llevar" y mirar una tarjeta: **verificar** que su total es exactamente el
   mismo que antes de esta corrección (FR-005).
7. En cualquier tarjeta de Domicilio: **verificar** que se ve **un solo total combinado**, sin
   línea aparte de "productos" y "domicilio" (FR-004).
8. Abrir la pestaña de red de DevTools y repetir el paso 1: **verificar** que abrir "Domicilios"
   **no** dispara una petición `checkout-preview` por cada tarjeta (Assumption de spec.md).

---

## Historia 2 — Crear pedido desde cualquier pestaña, con el tipo correcto y un botón claro (P1)

1. En "Mesas", "Domicilios" y "Para llevar" (sin tarjeta seleccionada), en los tres anchos:
   **verificar** que el botón de crear pedido está **visible y habilitado** en las tres pestañas
   (FR-007, escenarios 1–2, 10).
2. En **móvil**: **verificar** que el botón muestra un **texto** ("Crear pedido" / "Crear pedido
   nuevo"), no solo el ícono `+` (FR-013, SC-003).
3. En **tablet** sin tarjeta seleccionada: **verificar** que el botón aparece como acción fija
   sobre la grilla/lista, con etiqueta de texto visible (FR-015, escenario 10).
4. En **escritorio**: **verificar** que el botón sigue fijo al final del panel de bienvenida con su
   etiqueta completa "+ Crear pedido nuevo" (FR-015, escenario 9).
5. Desde "Domicilios", pulsar el botón: **verificar** que el armado manual abre con tipo
   **Domicilio** preseleccionado (FR-008, escenario 3).
6. Volver, desde "Para llevar", pulsar el botón: **verificar** que abre con tipo **Para llevar**
   preseleccionado (FR-009, escenario 4).
7. Desde "Mesas" (o desde el panel sin selección), pulsar el botón: **verificar** que abre **sin**
   tipo preseleccionado — como hoy (FR-010, escenario 5).
8. Con un tipo preseleccionado, cambiar el tipo dentro del formulario: **verificar** que el cambio
   se acepta (FR-011, escenario 6).
9. Crear el pedido: **verificar** que a dónde navega, cómo se guarda y qué permisos exige son los
   mismos que hoy (FR-012, escenario 7).
10. Con una mesa libre seleccionada, cambiar a "Domicilios" y pulsar crear: **verificar** que abre
    tipo Domicilio y la mesa previa se resuelve con el criterio ya existente (Edge Case).

---

## Historia 3 — Panel de detalle completo en cualquier ancho (P1)

Para cada ancho (escritorio ~1280px, tablet ~900px, móvil ~390px), con un pedido seleccionado:

0. (Antes de seleccionar, en **tablet ~900px**) **Verificar** que sin ninguna mesa ni pedido
   seleccionado no hay panel de bienvenida permanente: la grilla ocupa todo el ancho y el botón de
   crear pedido va como acción fija sobre ella (FR-016a).
1. **Verificar** que **todo** el contenido del panel (encabezado, lista de productos, totales,
   selector de método de pago, botones de acción) queda dentro del área visible — nada recortado
   por el borde ni fuera de la pantalla (FR-016, FR-017, SC-004).
2. **Verificar** que la **página no tiene desplazamiento horizontal** en ningún ancho (FR-018,
   SC-004).
3. En **tablet**: **verificar** que al seleccionar, el panel **reemplaza** la grilla a todo el
   ancho; al cerrar el detalle, se vuelve a la grilla en el **mismo estado** — misma pestaña, mismo
   filtro, mismo scroll (FR-016 esc. 2).
4. Con un pedido más alto que el espacio disponible: **verificar** que se desplaza **verticalmente
   dentro del panel**, nunca en horizontal, y nada queda recortado sin forma de alcanzarlo
   (FR-019).
5. Con nombres de producto largos / dirección larga / nombre de cliente largo: **verificar** que
   ese texto envuelve/se acomoda al ancho del panel sin ensancharlo (FR-020).
6. **Verificar** que todos los controles de cobro se pueden pulsar en los tres anchos (FR-021).

---

## Historia 4 — Varios productos a la vez en el panel de detalle (P2)

1. Seleccionar el pedido de ≥ 6 productos.
2. En **escritorio** y **tablet**: **verificar** que se ven **al menos 4 productos a la vez** sin
   desplazarse, y que la lista ocupa el alto vertical restante (no una franja de 1–2) (FR-022,
   SC-005).
3. En **móvil**: **verificar** que se ven al menos 2 productos y que los botones de cobro siguen
   alcanzables sin desplazar toda la pantalla (FR-025, SC-005).
4. Desplazarse por la lista: **verificar** que el scroll ocurre **dentro del área de la lista**,
   con el encabezado del pedido y la zona de totales/acciones siempre visibles (FR-023).
5. Seleccionar un pedido de 1–2 productos: **verificar** que la lista no fuerza un alto artificial
   ni deja un hueco vacío con aspecto de error (FR-024).

---

## Historia 5 — Información del domicilio compacta, junto al estado (P2)

1. Seleccionar el pedido de Domicilio con dirección larga.
2. **Verificar** que dirección + teléfono + valor del domicilio se muestran en una **fila compacta
   junto a la insignia de estado del pedido**, no como un bloque vertical separado y extenso
   (FR-026).
3. **Verificar** que los tres datos siguen completos y legibles — no falta ninguno (FR-027).
4. **Verificar** que la dirección larga se muestra **completa, envolviendo en 2–3 líneas**, sin
   truncarse ni esconderse tras una interacción (FR-028).
5. **Verificar** que el valor del domicilio de esa fila es el **mismo** que está sumado en el total
   del pedido (Historia 1) — nunca dos cifras distintas (FR-029).
6. **Verificar** que el alto que antes ocupaba ese bloque ahora está disponible para la lista de
   productos (SC-006) — comparar con una captura previa si es posible.
7. Seleccionar un pedido de mesa o "Para llevar": **verificar** que **no** aparece esa fila de
   información de domicilio (FR-030).

---

## Historia 6 — Menú de navegación colapsado en tablet (P2)

1. Confirmar que `A-71` está registrada (prerrequisito).
2. En **tablet (~900px)**, abrir varias pantallas de la app (Terminal de Mesas, Órdenes, Inventario,
   Reportes…): **verificar** que el menú de navegación global está **oculto por defecto** y no
   reserva ancho fijo permanente (FR-031, FR-032).
3. Pulsar el control de menú (el mismo botón/hamburguesa que en móvil): **verificar** que el menú
   se despliega **superponiéndose** al contenido, con las mismas opciones que en escritorio/móvil
   (FR-031, FR-035, FR-036).
4. Elegir una opción, o tocar fuera del menú: **verificar** que el menú se vuelve a ocultar (mismo
   comportamiento que móvil).
5. En la Terminal de Mesas en tablet con el menú oculto: **verificar** que el ancho liberado queda
   disponible para la grilla y el panel de detalle (FR-033, SC-007).
6. En **escritorio (~1280px)**: **verificar** que el menú sigue mostrándose **fijo como hoy**
   (FR-034) — este cambio no afecta el escritorio.
7. Rotar la tablet cruzando el umbral de 1024px (o cambiar el ancho de DevTools de 900px a 1100px):
   **verificar** que el menú pasa de forma consistente al comportamiento del nuevo ancho, sin
   perder la selección ni el estado de la pantalla (Edge Case).
8. Desplegar el menú en tablet con un pedido seleccionado en la Terminal: **verificar** que el menú
   se superpone sin cerrar la selección ni el panel de detalle (Edge Case).

---

## Verificación cruzada de no regresión

1. **Grilla y barra superior (spec 076)**: en los tres anchos, la grilla responsive de mesas
   (4/3/2 columnas), la barra superior operativa, los contadores de ocupación y "Atendido por" se
   ven y se comportan igual que antes de esta spec (FR-037).
2. **Vocabulario y colores**: las insignias de estado ("Por confirmar", "En preparación", "Listo",
   "Cobro pendiente"…) usan las mismas etiquetas y los mismos colores (FR-038).
3. **Acciones**: buscar mesa (F2), filtrar por ocupación, seleccionar, crear pedido (F3), cobrar y
   liberar mesa siguen disponibles y funcionando en escritorio, tablet y móvil (FR-040, SC-008).
4. **Cobro**: cobrar un pedido de Domicilio emite la misma venta/factura que antes — el total
   facturado no cambia (FR-039); esta spec solo alineó lo que **muestra** la tarjeta.
5. Correr `ng test --watch=false` completo: **verificar** que toda la suite pasa, incluidos los
   `*.spec.ts` de las áreas tocadas.
