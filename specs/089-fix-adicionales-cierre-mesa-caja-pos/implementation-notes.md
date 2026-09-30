# Notas de implementación — Spec 089

**Fecha de inicio**: 2026-09-30 · **Ramas**: `feat/089-qr-addons-per-line` en `pos-backend` y `pos-heladeria`

## Línea base (T004) — antes de tocar código

| Suite | Comando | Resultado |
|---|---|---|
| `pos-backend` | `python -m unittest discover -s app/characterization_tests -p 'test_*.py'` (venv `env/`) | **1127 tests, OK** |
| `pos-heladeria` | `npx ng test --watch=false` | **1033 tests: 1013 OK, 19 fallos, 1 omitido** (6 archivos con fallos) |

Nota: el runner de `pos-heladeria` es Vitest (`@angular/build:unit-test`), no Karma; la opción
`--browsers=ChromeHeadless` que citan las tareas no aplica y falla por falta de un proveedor de
navegador. Se corre sin `--browsers`.

Los **19 fallos son previos y ajenos** a esta spec (los mismos de la spec 088): `app.spec.ts` (2),
`auth.service.spec.ts` (8), `menu.service.spec.ts` (1), `tenant.service.spec.ts` (6),
`pos-checkout-panel.component.spec.ts` (1: "T032: ofre…"), `pos-order-panel.component.spec.ts` (2).
Cualquier fallo fuera de esa lista es una regresión.

## Registro de avance

### Fase 2 — Esquema (T005–T009)

- `QR_ADDONS_PER_LINE` añadido a `Settings` (default `True`). Migración `a89f0c1d2e3b_089_line_addons.py`
  (`down_revision = deb68185a606`), modelos `CartItem/OrderItem.addons_total`, `CartItemOption/OrderItemOption.per_line`,
  `OrderItem.line_total`. Test `test_line_addons_migration.py` (6 tests, OK).
- **T009 verificado en PostgreSQL 16** sobre una copia desechable de la BD de desarrollo (`pos_test_089`, ya eliminada;
  `pos_db` no se tocó): `upgrade head` añade las 4 columnas (`numeric NOT NULL default 0` / `boolean NOT NULL default false`)
  y los 2 `CHECK`; el esquema `heladeria` tenía 57 `order_items` con total `unit_price×quantity` = 855.000,00 antes y después,
  con 0 filas `addons_total<>0`; `downgrade -1` corre sin error con 0 líneas con adicionales; `upgrade head` otra vez OK;
  con una línea con `addons_total = 100` el `downgrade` **aborta** con el mensaje que apunta a `QR_ADDONS_PER_LINE=false`.
- Observación: `tenant_default` es un esquema de plantilla sin tablas de negocio; la migración lo omite con `_has_table`.


### Fase 3 — US1 adicionales por línea (T010–T039)

- Motor: `catalog_engine/core.py` (`ChosenOption.per_line`, `compute_unit_price`, `compute_addons_total`, `line_total`; `compute_line_price` conserva firma y resultado) y `consumption.py` (`per_line` ⇒ `per_unit × chosen.quantity`).
- `chosen_from_rows` vive en `orders/consumption.py` (no en `checkout.py`, como decía T018) y se reexporta/usa desde `checkout.py`, `kitchen.py` y `consolidation.py`; reemplaza las dos copias de `_item_options`. `sales/consumption.py::_sale_item_options` lee `per_line` del snapshot JSONB.
- `cart/service.py`: `_mark_addons` (solo grupos `con_recargo`, solo si `QR_ADDONS_PER_LINE`), `_chosen_from_cart_rows`, `serialize_cart`/`submit_cart` con `line_total`; `update_item` sin `options` conserva las marcas de la fila (una línea histórica sigue histórica al cambiar solo la cantidad).
- `orders/schemas.py` expone `addons_total`, `line_total`, `per_line`; `SaleLine.addons_total`; `set_assignments` deja los adicionales solo en la fila original (A-97 anotada sin corregir).
- Script `app/scripts/fold_line_addons.py` (simulación por defecto) con `test_fold_line_addons.py`.
- Frontend: helpers `lineTotal`/`lineTotalGross` (`dining.interface.ts`); `ProductSelect.addonsPerLine`, `CartLine.addonsTotal`, store/paneles/menú público migrados a `lineTotal`.
- **Desviaciones**: (1) `consolidate_table` **no** copia `combo_id` (T013 lo pedía; hoy no lo copia y los combos están retirados desde la spec 063: no se cambia comportamiento ajeno, Principio V). (2) El frontend usa Vitest, no Karma: `ng test --watch=false` sin `--browsers`. (3) `Option` del menú público no trae `pricing_type`: en el pie del selector basta con que los grupos «incluidos» cuestan $0.
- **T029**: `tables_advanced.group_bill` ya pasa por `SaleLine.line_total`; sin cambio de código.

### Fase 4 — US2 total inmediato (T041–T048)

- Backend: `InsufficientStockError.item_name`, `sold_out_detail` (`catalog_engine/consumption.py`) y `deduct_order_items_naming_product` (`orders/consumption.py`) usado por `create_order`, `add_item_to_order` y el reemplazo de `kitchen.py`. Status 400 conservado; los demás llamadores de `deduct_order_items` no cambian.
- Frontend: `checkoutPreviewEstimate` en `PosTerminalStore`; `saveOrder` (estimado → confirmado → revierte, serializado con `submitting`; las líneas guardadas antes de un fallo salen del borrador para que reintentar no las duplique) y `voidPersistedItem`; `pos-checkout-panel`, `payment-attempt-review-panel` y `manual-order-page` sin `confirm.ask` (aviso no bloqueante + segundo clic). Se retiró `totalChangeAck`.
- **T045b**: en el frontend no existe hoy un «reemplazo de línea»; solo se cubrió `voidPersistedItem`.
- SC-010: `grep "title: 'El total cambió'" src` → 0 coincidencias.

### Fase 5 — US3 editar adicionales (T050–T055)

- Backend: `update_item` ya cumplía (T051: sin cambios); se congela con 10 tests (`TestQrEditAddons`).
- Frontend: `DiningCartService.updateItem`, `CartLine.optionSelections`, botón «Editar adicionales» (≥ 44 px) en `cart.component`, y edición en `public-menu` reutilizando `ProductSelect` (`quantityLocked`, `staleOptionNames`, `externalError`, fila «No disponible» con «Quitar» que bloquea guardar). Un adicional desactivado ya no viene en el menú: se avisa y se quita al guardar.

### Fase 6 — US4 cierre de mesa (T057–T065)

- `try_release_if_empty` devuelve `list[TableSession]`; `notify_sessions_closed` (best-effort, tras el commit); `checkout.release_table(closed_out=…)` + `orders/router.py` (`released`); `cart/service.py::leave_session(tenant_id)` y `cancel_my_order` (`empty`); `qr_context._abandon_expired` (`empty`) y cabecera `X-Session-State: closed` en los dos 401 de cierre; CORS `expose_headers`.
- Frontend: `DinerSessionExpiredError.closed`, vista `'closed'` con el texto exacto, `expireSession('closed'|'expired')`, `handleSessionError`, cierre de overlays.
- Se ajustó `app/scripts/test_table_release.py` (`bool(try_release_if_empty(...))`).

### Fase 7 — US5 impresión (T067–T069) — causa real

Reproducida con **Chrome real** (headless, CDP: `Emulation.setEmulatedMedia print` + `Page.printToPDF`) sobre el reporte de un turno cerrado de una copia de la BD de desarrollo, luego renderizada con poppler/ghostscript. **No era ninguna de las tres hipótesis de research D13**, aunque la 1 también aplicaba:
1. **Hoja en blanco (causa principal)**: el navegador pagina a ~816 px de ancho (< `lg`), así que el telón del menú móvil `fixed inset-0 z-30 lg:hidden` (renderizado porque `sidebarOpen()` es true en escritorio) deja de estar oculto y se pinta a página completa **encima** del reporte en cada hoja. Se reproduce también con un turno sin movimientos. Bisección: aislar `app-cash-report` imprime bien; quitar sidebar/header no arregla; cualquier CSS que colapse el telón (`app-dashboard-layout > div {height:auto}`) sí.
2. **Sin paginar**: `h-screen overflow-hidden` + `main overflow-y-auto` recortan lo que excede la primera hoja.
3. Colores: el tema oscuro no existe en la app (cero `dark:`); aun así se fuerzan negro sobre blanco en impresión.
Corrección: `print:hidden` en el telón; en `dashboard-layout` `.shell-root/.shell-content/.shell-main` liberados en `@media print` acotado a `body.printing-cash-report`; barra superior del módulo de caja `print:hidden`; `cash-report` con negro/blanco forzado, `print-color-adjust: exact`, `break-inside: avoid` en filas y `thead` repetido.
Verificación (Chrome headless, `printBackground:false`): turno cerrado → 1 hoja con contenido; emulando modo oscuro automático → igual; turno inflado a 60× filas → 19 hojas, **ninguna en blanco**. **Firefox y la vista previa real de Chrome siguen sin comprobarse (T070 abierta).**

### Fase 8 — verificación final (T071–T075)

- Backend: **1217 tests, OK** (base 1127: +90).
- Frontend: **1093 OK, 19 fallos, 1 omitido** (base 1013 OK / 19 fallos): los mismos 19 previos y ajenos (mismos archivos y conteos); `ng build` sin errores.
- Verificación en vivo contra PostgreSQL (copia desechable de la BD de desarrollo, ya eliminada; `pos_db` no se tocó): sesión QR → 2 × $15.000 + adicional $1.000 = **$31.000** (`addons_total=1000`, `per_line=true`); cantidad 3 → $46.000; sin adicionales → $45.000; adicional x2 → $47.000; «Liberar mesa» → el token del comensal recibe **401 con `X-Session-State: closed`** (cuerpo intacto) al leer pedidos y al agregar al carrito.
- Trazabilidad: A-94…A-97 presentes en `registro-de-anomalias.md`; los `CONGELA` modificados citan A-94/A-96.

### Fase 9 — Convergencia: acceso denegado y fin de acceso único (T076–T080, T082)

- `DinerTokenStore.endAccess(tableToken)`: único borrado de fin de acceso (token, toda clave `pos.diner.*` de `localStorage` y `sessionStorage`, cookies `pos.diner*`) que deja **solo** `pos.diner.exited_token` = token público en `sessionStorage`. Falla en silencio con almacenamiento bloqueado.
- `public-menu.component.ts`: `exit()` y `expireSession('closed')` comparten `endAccess` y terminan en `'closed'` (gracias); un evento o 401 tardío con la vista ya en `'closed'` no la cambia. `ngOnInit` con 401 `closed` (token anterior en `?s=`/`localStorage`) → `denyAccess()` → `'exited'`; 401 sin cabecera conserva `'name'`. `'exited'` es ahora la pantalla única `data-testid="vista-acceso-denegado"` con el texto "Por favor, escanea nuevamente el código QR de la mesa para ingresar al menú" (se acabó "Acceso finalizado…"). El vencimiento tardío (`'expired'`) sigue respetando la marca.
- Refactor interno: `resetSessionState()` (sondeo, SSE, carrito, pedidos, overlays) compartido por `expireSession` y `denyAccess`.
- Pruebas: `diner-token.store.spec.ts` (+4) y `public-menu.component.spec.ts` (reescritos los casos de "Salir", 401 al cargar y texto; nuevos: estado del navegador idéntico para `paid|swept|released|empty` y "Salir", recarga ⇒ acceso denegado con **cero** `openSession`, pestaña sin marca ⇒ nombre, `sessionStorage` bloqueado ⇒ gracias sin lanzar). Frontend: **1101 OK, 19 fallos previos ajenos, 1 omitido** (antes 1093 OK); `ng build` sin errores.
- FR-017a en backend: ya cubierto por `test_table_sessions_router.py` (sesión cerrada sin reocupar y reocupada ⇒ 401 con `X-Session-State: closed`); sin cambio de código.
- T082: `research.md` D13 completado con la causa confirmada de la hoja en blanco.
- **T081 sigue abierta** (recorrido manual con teléfono/DevTools: marca en `sessionStorage`, F5/Atrás/Adelante sin `POST` de apertura, `?s=` viejo con la mesa libre/cerrada/reocupada).

### Pendiente

- **T002**: confirmar con el negocio (a) aviso no bloqueante + segundo Cobrar y (b) 401 por vencimiento → pantalla de nombre. **Bloquea el despliegue.**
- **Recorridos manuales** (T040, T049, T056, T066, T070, T073, T081): el navegador de pruebas se desconectó a mitad de la sesión, así que no se recorrieron en la UI: comanda/inventario en cocina, SC-008 (< 0,2 s), SC-005/SC-006 con teléfono real y SSE, Firefox/vista previa de impresión.
- **Ramas**: todo el trabajo está sin commit en `feat/089-qr-addons-per-line` (backend y frontend). El plan pedía separar US2/US4 y US5 en `fix/089-table-close-pos-total-print` y `fix/089-cash-report-print`: se divide al commitear (commits pequeños en inglés, migración primero y sola).
- **Orden de despliegue**: backend con migración primero, luego frontend; una vez existan líneas con `addons_total > 0` la reversa es `QR_ADDONS_PER_LINE=false`, no el downgrade.
- La BD de desarrollo `pos_db` **aún no tiene la migración** `a89f0c1d2e3b` (`alembic upgrade head`).
