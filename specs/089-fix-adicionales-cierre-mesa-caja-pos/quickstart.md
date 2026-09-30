# Quickstart — Validación de la spec 089

**Spec**: [spec.md](./spec.md) · **Contratos**: [contracts/](./contracts/) · **Modelo**: [data-model.md](./data-model.md)

Guía para comprobar de extremo a extremo que cada historia funciona. Los detalles de implementación viven en `tasks.md`.

## 0. Prerrequisitos

- `pos-backend` y `pos-heladeria` en las ramas `feat/089-qr-addons-per-line`, `fix/089-table-close-pos-total-print` (creada desde la anterior) y `fix/089-cash-report-print` (Principio XIV; ver T003), con la migración de la spec aplicada (`alembic upgrade head`; §1).
- Un negocio de pruebas con: producto **Hamburguesa** (presentación única $15.000) con un grupo **con recargo** "Adicionales" (opción **Tocino** $3.000, con insumo), un grupo **incluido** "Sabor" (opción con insumo), una mesa libre con QR, un turno de caja abierto, un método de pago por transferencia (con QR) y stock suficiente. Producto **Gaseosa** $5.000.
- Backend con Redis levantado (SSE) y `QR_ADDONS_PER_LINE` sin definir (por defecto activa).
- Chrome y Firefox, más un teléfono (o emulación móvil) para el Menú QR.

Comandos de verificación automática:

```bash
# pos-backend
python -m unittest discover -s app/characterization_tests -p 'test_*.py'
# pos-heladeria
npx ng test --watch=false --browsers=ChromeHeadless
```
Ambas suites completas deben quedar en verde (SC-003, regresión).

## 1. Migración (Historia 1, Principio VIII)

1. Antes: `SELECT count(*) FROM tenant.order_items;` (anotar el valor y un total de venta conocido).
2. `alembic upgrade head` ⇒ 4 columnas nuevas; `\d tenant.order_items` muestra `addons_total numeric(12,2) not null default 0`.
3. Después: mismo conteo; `SELECT count(*) FROM tenant.order_items WHERE addons_total <> 0` = 0; el total de la venta conocida **no cambia** (SC-003).
4. `alembic downgrade -1` corre sin error solo si no hay líneas con `addons_total > 0` (en cuanto exista un pedido con adicionales, la reversa soportada es `QR_ADDONS_PER_LINE=false`, no el downgrade); `upgrade head` otra vez.

## 2. Historia 1 — Adicionales por unidades elegidas (SC-001, SC-002)

En el Menú QR (móvil):
1. Agregar **2 hamburguesas** con **1 Tocino** ⇒ la línea marca **$33.000** y el detalle "Tocino x1"; el carrito, **$33.000**.
2. Subir la cantidad a **3** ⇒ **$48.000**; el tocino sigue en x1.
3. Con un grupo con selector de cantidad, poner **2** unidades de un adicional en 1 hamburguesa ⇒ $21.000 (2 × $3.000 una sola vez).
4. Producto en promoción con adicional ⇒ total = (precio promocional × cantidad) + adicionales.
5. Enviar el pedido (transferencia con comprobante) y comparar el **mismo total** en: terminal POS (tarjeta del pedido), panel de cobro, "Pagos por confirmar" y, tras cobrar, el detalle de venta. (SC-002)
6. Comanda: "2 × Hamburguesa · Tocino x1". Inventario del insumo del tocino: descuenta **1** (no 2); el de sabores/receta, por unidad.
7. Cancelar/anular el pedido antes de preparar ⇒ el kardex devuelve **1** de tocino (deduct == reverse).
8. Un pedido anterior al despliegue conserva total, comanda y descuento exactos.
9. `QR_ADDONS_PER_LINE=false` + reinicio ⇒ una línea nueva vuelve a la regla por unidad; la del paso 1 ya creada sigue en $33.000.

## 3. Historia 3 — Editar adicionales (SC-004)

1. En el carrito, pulsar **Editar adicionales** en la línea de 2 hamburguesas + tocino ⇒ se abre el selector con Tocino y la nota ya elegidos y botón **Guardar cambios**.
2. Cambiar el tocino a x2 → guardar ⇒ misma línea, misma cantidad (2), total $36.000. Contar interacciones: abrir, modificar, guardar = 3.
3. Quitar todos los adicionales → guardar ⇒ la línea queda sin adicionales y sin eliminarse ($30.000).
4. Abrir y **cancelar** ⇒ la línea intacta y ninguna llamada al backend.
5. Violar un mínimo/máximo del grupo ⇒ no permite guardar y dice qué corregir.
6. Enviar el pedido ⇒ la línea enviada ya no ofrece "Editar adicionales".
7. Dos teléfonos en la misma mesa: cada uno solo ve/edita su carrito.
8. Desactivar el adicional en el catálogo con la línea en el carrito ⇒ al editar aparece "no disponible" y no se puede guardar sin quitarlo/reemplazarlo.

## 4. Historia 4 — Cierre de mesa (SC-005, SC-006)

Con un teléfono conectado a la sesión (con algo en el carrito), cerrar la mesa desde el POS **por cada camino** y anotar el tiempo hasta la pantalla:

| # | Camino | Preparación |
|---|---|---|
| a | Cobro completo de la mesa | pedido enviado y cobrado desde el POS |
| b | "Liberar mesa" (cajero) | mesa sin órdenes sin cerrar |
| c | "Liberar mesa" (mesero) | ídem, con usuario mesero |
| d | Barrido del scheduler | forzar `TABLE_SESSION_MAX_HOURS`/ventana vacía en el entorno de pruebas |
| e | Cierre automático de sesión sin órdenes | un solo comensal sale (`Salir`) o vence sin pedir |

Esperado en a, b, c y e (< 5 s, aviso en tiempo real). En d, si la mesa tiene pedidos por cobrar, el barrido no cierra la sesión ni emite aviso: la pantalla aparece en el siguiente sondeo (≤ 10 s con la pestaña visible) o al reusar la sesión. Pantalla: "**¡Gracias por tu visita! Esperamos verte pronto.**", sin historial/recibo/botón; carrito local vacío. Estado del navegador (DevTools → Application): sin `pos.diner.session_token`, sin `pos.diner.checkout_progress.*`, sin cookies del comensal; en `sessionStorage` **solo** `pos.diner.exited_token` = token público de la URL de la mesa.

Además:
1. Tras (a–e), en las herramientas de red reintentar `POST /cart/submit` con el token viejo ⇒ **401** con `X-Session-State: closed`.
2. Teléfono sin conexión al cerrar (modo avión) → cerrar la mesa → reconectar / recargar ⇒ primera petición 401 ⇒ pantalla de gracias (SC-006).
3. Reabrir/ocupar la mesa en el POS ⇒ el teléfono anterior **sigue** en la pantalla de gracias; escanear el QR de nuevo ⇒ flujo de nombre y sesión nueva.
4. Varios comensales conectados ⇒ todos ven la pantalla.
5. Transferencia enviada sin revisar y cierre de mesa ⇒ el comensal ve gracias y el pago pendiente sigue en "Pagos por confirmar".
6. Un comensal que salió con **Salir** antes del cierre sigue en su estado de acceso cerrado, sin duplicarse.
7. **Recarga tras el cierre** (F5, "Atrás", "Adelante" en la misma pestaña, SC-011) ⇒ pantalla estática "Por favor, escanea nuevamente el código QR de la mesa para ingresar al menú", sin campo de nombre ni botón; en la pestaña de red **no** hay `POST` de apertura de sesión. Repetir para cada camino a–e.
8. **Misma huella que "Salir"**: comparar el navegador tras el cierre con el de otro teléfono que pulsó "Salir" ⇒ idéntico (mismas claves, solo la marca mínima); recargar este último ⇒ la misma pantalla y el mismo texto.
9. **Pestaña nueva / escaneo nuevo** del QR tras el cierre ⇒ flujo de nombre y sesión nueva (la marca es por pestaña).
10. **Token anterior** (`?s=<token viejo>` o `localStorage` con el token) con la mesa libre, cerrada y reocupada ⇒ `401` con `X-Session-State: closed` y pantalla de acceso denegado.
11. `sessionStorage` bloqueado (modo privado estricto) ⇒ el cierre igualmente muestra gracias y no lanza error; la recarga pide nombre (limitación aceptada) y el token viejo sigue en `401`.

## 5. Historia 2 — Total inmediato (SC-008, SC-009, SC-010)

En la terminal POS, con una orden abierta de $20.000 (repetir en **mesa**, **domicilio** y **para llevar**):
1. Agregar **1 Gaseosa $5.000** ⇒ el total visible pasa a **$25.000** en < 0,2 s (con "actualizando…"), Cobrar deshabilitado un instante, luego habilitado con la cifra confirmada. **Ningún diálogo.**
2. Elegir el método de pago y pulsar **Cobrar** ⇒ venta por $25.000 sin "El total cambió".
3. Agregar tres productos seguidos ⇒ total final = suma correcta, sin cifras intermedias colgadas.
4. Simular estimación distinta del servidor (producto con promoción) ⇒ la cifra se reemplaza por la del servidor sin diálogo y se cobra la confirmada.
5. Agotar el insumo de un producto y agregarlo ⇒ mensaje "«Producto · Presentación» está agotado: falta <insumo>"; el total vuelve al de antes (SC-009).
6. Cambio externo (modificar la orden desde otro dispositivo o vencer una promoción entre el preview y Cobrar) ⇒ primer Cobrar: aviso no bloqueante en el panel y **sin cobro**; segundo Cobrar cobra el importe visible.
7. Detener el backend justo tras agregar ⇒ error habitual, total revertido/"Calculando el total…", sin cobro (FR-031).
8. "Pagos por confirmar" con un pago QR pendiente: mismo comportamiento (sin modal; cambio ⇒ aviso y segundo clic).
9. Pedido manual: crear un pedido con un borrador que cambió de total ⇒ aviso y segundo clic, sin modal.
10. Anular una línea de la orden (tras confirmar "¿Anular este ítem?") ⇒ el total baja al instante y luego queda el confirmado; si el backend falla, vuelve al total anterior. Medir en las herramientas de rendimiento del navegador que el total estimado aparece en < 0,2 s (SC-008).
11. Búsqueda global en `pos-heladeria/src`: `grep -rn "El total cambió"` solo debe encontrar textos del **aviso no bloqueante** y la barra `checkoutPreviewStale`, ningún `confirm.ask` con ese título (SC-010).

## 6. Historia 5 — Impresión del cierre de caja (SC-007)

Primero **reproducir la hoja en blanco** con el código actual (FR-024) y registrar la causa en `implementation-notes.md`; después corregir y repetir:

1. Cerrar un turno con movimientos ⇒ **Imprimir / Exportar reporte**.
2. En la **vista previa de impresión** de Chrome **y** de Firefox: contenido completo, negro sobre blanco, sin menú lateral/encabezado/botones, sin hojas en blanco. Con **tema claro y oscuro**.
3. Turno con muchos movimientos (≥ 2 hojas): todas las hojas con contenido y sin hoja vacía intermedia.
4. Turno sin movimientos ⇒ el reporte imprime encabezados y totales en cero, no una hoja vacía.
5. "Guardar como PDF" ⇒ nombre `<slug-del-negocio>-<DD-MM-YYYY>` (spec 087).
6. Imprimir un recibo de venta, un recibo de mesa y la hoja de QR ⇒ **sin cambios** respecto a antes (FR-023).

## 7. Criterios de salida

| SC | Cómo se comprueba |
|---|---|
| SC-001, SC-002 | §2 pasos 1–5 |
| SC-003 | §1 + suites completas en verde |
| SC-004 | §3 paso 2 |
| SC-005, SC-006 | §4 |
| SC-007 | §6 |
| SC-008–SC-010 | §5 |
