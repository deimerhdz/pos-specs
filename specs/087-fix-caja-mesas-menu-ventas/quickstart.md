# Quickstart: Correcciones de Caja, Terminal de Mesas, Menú QR y Ventas

**Spec**: [spec.md](./spec.md) | **Contratos**: [contracts/api-changes.md](./contracts/api-changes.md) | **Datos**: [data-model.md](./data-model.md)

Guía de validación manual/end-to-end por historia de usuario. No sustituye los tests
automatizados (unitarios, de integración, characterization) que se definen en `tasks.md`;
sirve para confirmar en un entorno real que el comportamiento observable coincide con los
criterios de aceptación de `spec.md`.

## Prerrequisitos

1. `pos-backend`: `docker compose up -d` (Postgres 16 + Redis + API) y
   `alembic upgrade head` aplicado (incluye las migraciones nuevas de esta spec: columnas de
   `customer_orders` y `DROP TABLE cash_partial_counts`).
2. `pos-heladeria`: `npm install` (si aplica) y `npm start` (`ng serve`), apuntando al backend
   local.
3. Un tenant de prueba con: al menos un producto con presentación y promoción activa (para el
   caso "Granizado de Mora"), un cajero con sesión iniciada, y una caja (`cash_register`) sin
   turno abierto al empezar (para poder abrir uno limpio y verificar el reinicio de
   numeración).
4. Rama de trabajo creada según Principio XIV de la constitución (`fix/087-...`) antes de
   tocar código en cualquiera de los dos repos.

---

## Historia 1 — Cobro correcto de adicionales sobre promociones (P1)

**Setup**: producto "Granizado de Mora" con precio promocional $5.000, adicionales "Leche
condensada" ($2.000) y "Choco-chips" ($1.500).

1. Desde el Menú QR (o Terminal), arma un pedido con ese producto + ambos adicionales.
   **Esperado**: total mostrado antes de enviar = **$8.500**.
2. Envía el pedido y, como cajero, ábrelo desde "Pagos por confirmar".
   **Esperado**: el total ahí coincide exactamente con $8.500.
3. Cobra el pedido y abre su detalle en el módulo de Ventas.
   **Esperado**: el total registrado es $8.500.

**Nota de alcance (ver `research.md` D6)**: este escenario debe probarse creando el pedido
**después** de aplicado el fix. No usar como caso de prueba una venta ya finalizada antes del
despliegue del fix — esas conservan su total histórico sin recalcular, por el Principio VII
de la constitución.

## Historia 2 — Consolidación de pedidos abiertos sin duplicados (P1)

1. Crea un pedido de mesa con un producto. Anota su `order_id` y total.
2. Desde el detalle de ese pedido (no pagado), usa "Agregar producto", añade un ítem, guarda.
   **Esperado**: el total de la mesa aumenta, el pedido conserva el mismo `order_id`/número —
   no aparece un segundo pedido nuevo en la lista.
3. Repite con un pedido "para llevar" y uno "a domicilio" abiertos.
   **Esperado**: mismo comportamiento — los ítems se anexan al pedido existente.
4. Cobra un pedido y luego intenta agregarle productos.
   **Esperado**: el sistema lo impide (ver contrato `POST /orders/{order_id}/items` → `409`
   en `contracts/api-changes.md`) y guía a crear un pedido nuevo.

## Historia 3 — Numeración estable y cronológica de pedidos de mesa (P2)

**Setup**: abre un turno de caja nuevo (sin pedidos de mesa previos en él).

1. Crea el Pedido A de mesa a una hora `T`. **Esperado**: se muestra como `#1`.
2. Cinco minutos después, crea el Pedido B en otra mesa. **Esperado**: A sigue en `#1`, B en
   `#2` (A no se desplaza).
3. Crea un tercer pedido C. **Esperado**: C es `#3`; A y B no cambian.
4. Cierra el turno y abre uno nuevo; crea un pedido de mesa D.
   **Esperado**: D es `#1` (el contador se reinició con el nuevo turno — FR-006).

Verificación directa en base de datos (opcional, para depurar): `SELECT id, table_order_number, cash_shift_id, created_at FROM tenant.customer_orders WHERE order_type = 'DINE_IN' ORDER BY created_at;`
— confirma que `table_order_number` no cambia entre lecturas sucesivas para la misma fila.

## Historia 4 — Cierre de caja limpio, sin Arqueo Parcial (P2)

1. Navega el módulo de Caja completo (dashboard, historial). **Esperado**: ningún botón,
   acceso ni registro consultable de "Arqueo Parcial" (confirma también que
   `POST /cash/shifts/{id}/partial-count` ya no existe: `curl -i -X POST .../partial-count`
   debe responder `404`).
2. Cierra el turno de un tenant llamado "Tenant de Prueba" el 28-09-2026 y presiona
   "Imprimir" en el resumen. **Esperado**: se abre el diálogo de impresión/guardar-PDF del
   navegador mostrando **solo** el resumen financiero (sin sidebar ni header visibles en la
   vista previa de impresión), con `tenant-de-prueba-28-09-2026` como nombre sugerido.
3. Repite con un tenant cuyo nombre tenga eñes/tildes/símbolos.
   **Esperado**: el nombre sugerido usa el slug limpio (alfanumérico + guiones).

**Riesgo conocido** (ver `research.md` D1): si el navegador no respeta `document.title` como
nombre sugerido de forma consistente, este paso debe registrarse como hallazgo y reevaluar la
alternativa de generación de PDF vía Blob documentada en `research.md`.

## Historia 5 — Nombre de cliente visible y obligatorio, pedidos en paralelo (P2)

1. Intenta guardar un pedido nuevo (mesa, para llevar o domicilio) sin nombre de cliente.
   **Esperado**: el guardado se bloquea hasta diligenciarlo (backend `422` si se intenta sin
   pasar por el formulario, ver `contracts/api-changes.md` §3).
2. Guarda un pedido con nombre. **Esperado**: el nombre aparece destacado en la tarjeta y en
   el detalle.
3. Sobre una mesa con un pedido abierto, usa "Crear pedido manual".
   **Esperado**: se crea un segundo pedido independiente sobre la misma mesa, con su propio
   número (`table_order_number` distinto) y total.
4. Abre un pedido histórico (creado antes de esta spec, sin nombre).
   **Esperado**: se muestra un placeholder ("Cliente sin nombre"), sin bloquear la consulta.

## Historia 6 — Adicionales, notas y presentación explícitos (P3)

1. Arma un pedido con un producto con presentación "16oz", el adicional "Queso extra" (x1) y
   "Choco-chips" (x2), más una nota.
   **Esperado**: el resumen muestra "Producto · 16oz", "Queso extra x1", "Choco-chips x2" y la
   nota, todos resaltados.
2. Repite sin adicionales ni notas. **Esperado**: no se muestra ningún contenedor vacío.
3. Verifica el mismo pedido en el historial de Órdenes y en el detalle de Venta tras cobrarlo.
   **Esperado**: la presentación sigue visible junto al nombre del producto en ambos lugares.

## Historia 7 — Desglose de cambio en el detalle de venta (P3)

1. Cobra una venta 100% en efectivo con monto recibido mayor al total.
   **Esperado**: el detalle de venta muestra "Cambio: $X" con el valor exacto.
2. Cobra una venta 100% con un método distinto a efectivo (tarjeta o QR).
   **Esperado**: no aparece la línea de Cambio.
3. Cobra una venta con pago dividido donde ninguna porción fue en efectivo.
   **Esperado**: tampoco aparece la línea de Cambio (mismo comportamiento que el caso 2).

---

## Verificación de la migración destructiva (Arqueo Parcial)

Antes de ejecutar `alembic upgrade head` en cualquier entorno con datos reales, confirmar que
existe un backup reciente de la base de datos — el borrado de `cash_partial_counts` es
intencionalmente irreversible (FR-001, ver `data-model.md` §2). No es un paso de la
validación funcional, es un prerrequisito operativo de despliegue.

---

## Adenda 2026-09-29 — Historias 8 a 11

## Historia 8 — Presentación siempre visible junto a la promoción (P2)

**Setup**: producto con presentaciones "8 oz" (promoción 2x1), "Mediano" (15%) y una tercera
sin promoción, y otra con nombre largo + promoción.

1. Abre el Menú QR **en un ancho de celular (~360px)** y entra al modal del producto.
   **Esperado**: cada fila de "Elige tu presentación" muestra el nombre completo como etiqueta
   principal ("8 oz", "Mediano") y la promoción como etiqueta secundaria debajo.
2. Revisa la fila sin promoción. **Esperado**: solo el nombre, sin etiqueta vacía.
3. Revisa el nombre largo + promoción. **Esperado**: el nombre no queda en "…"; la promoción
   baja a otra línea.
4. **Abre el modal desde el Menú QR del comensal (escaneando/abriendo el enlace del QR de una mesa),
   no solo desde la Terminal.** **Esperado**: cada fila muestra "Grande"/"Mediano"… como etiqueta
   principal, incluso las que tienen promoción (corrección 2026-09-29: antes solo se veía la
   promoción porque el nombre llegaba vacío).

## Historia 9 — Adicionales visibles en el detalle y pedidos de la mesa (P1)

1. En la Terminal de Mesas agrega a una mesa un producto con "Queso extra" (x1) y
   "Choco-chips" (x2), **sin guardar todavía**. **Esperado**: ambos aparecen debajo del
   producto como "Queso extra x1" y "Choco-chips x2".
2. Guarda y recarga la pantalla. **Esperado**: siguen apareciendo, con el mismo formato.
3. Desactiva el producto en el menú (o déjalo no disponible) y vuelve a abrir el pedido
   guardado. **Esperado**: los adicionales siguen apareciendo por su nombre, no como
   fragmento vacío ni omitidos.
4. Un producto sin adicionales. **Esperado**: sin contenedor vacío.

## Historia 10 — Total actualizado al editar un pedido manual (P1)

1. Abre un pedido manual con total conocido (ej. $10.000), agrega un producto de $4.000 y
   guarda. **Esperado**: "TOTAL ORDEN" pasa a $14.000 **sin recargar**.
2. Anula/quita un producto y guarda. **Esperado**: el total descuenta ese producto.
3. Repite con un producto en promoción. **Esperado**: el total refleja la promoción
   reevaluada y coincide con el que muestra el cobro.
4. Quita todos los ítems. **Esperado**: "TOTAL ORDEN" = $0.
5. Abre la consola del navegador durante 1–4. **Esperado**: sin errores.

## Historia 11 — Notas por producto legibles y destacadas (P2)

1. Crea un pedido con la nota "sin azúcar" en un producto y revísalo en el panel de la
   Terminal, la página de pedido manual y el historial del Menú QR.
   **Esperado**: texto ≥ 16px, semibold/bold, fondo de alto contraste, mismo aspecto en las tres.
2. Producto sin nota. **Esperado**: sin contenedor.
3. Nota larga (varias líneas). **Esperado**: se ajusta sin desbordar ni ocultar el nombre del
   producto.
4. La nota general del pedido (`order.notes`) **no** cambia de aspecto.
