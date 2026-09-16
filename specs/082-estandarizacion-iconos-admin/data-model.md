# Fase 1 — Modelo: Catálogo de Íconos

**Spec**: [spec.md](./spec.md) | **Plan**: [plan.md](./plan.md) | **Investigación**: [research.md](./research.md)

No hay entidades de negocio ni persistencia nueva en esta funcionalidad (es un cambio de
presentación exclusivo del frontend — ver spec.md, Assumptions). El único "modelo" relevante es el
catálogo de íconos: la tabla de correspondencia entre cada nombre semántico usado hoy (por el
componente SVG artesanal o por un emoji) y el ícono concreto de Material Icons (variante Outlined,
Decisión D1 de `research.md`) que lo reemplaza.

## Entidad: Ícono

| Campo | Descripción |
|---|---|
| `nombre semántico` | Identificador estable usado por las pantallas para pedir un ícono (p. ej. `mesa`, `editar`) |
| `glifo actual` | Cómo se representa hoy: nombre de `@switch` en `app-icon`, o el carácter emoji literal |
| `origen` | `svg-artesanal` (vía `app-icon`) o `emoji` |
| `ícono Material Icons` | Nombre de la ligadura Outlined que lo reemplaza |
| `módulo/archivo` | Dónde se usa hoy (para ubicar la tarea de migración) |
| `alcance` | `admin` (se migra) / `compartido` (se migra, ver research.md D6) / `fuera de alcance` (no se toca) |

## Catálogo — reemplazos del componente SVG artesanal (`app-icon`, dentro de alcance)

| Nombre semántico | Ícono Material Icons (Outlined) |
|---|---|
| dashboard | `dashboard` |
| sales | `point_of_sale` |
| reports | `assessment` |
| orders | `receipt_long` |
| tables | `table_restaurant` |
| sessions | `event_seat` |
| products | `shopping_bag` |
| categories | `category` |
| promotions | `local_offer` |
| option-groups | `tune` |
| units | `straighten` |
| inventory | `inventory_2` |
| suppliers | `local_shipping` |
| cash | `payments` |
| payment-methods | `credit_card` |
| users | `group` |
| eye | `visibility` |
| eye-off | `visibility_off` |
| settings | `settings` |
| tenants | `storefront` |
| home | `home` |
| receipt | `receipt` |
| cart | `shopping_cart` |
| exit | `logout` |
| search | `search` |
| layers | `layers` |
| transfer | `sync_alt` |
| upload | `upload` |
| close | `close` |
| back | `arrow_back` |
| check-circle | `check_circle` |
| alert-circle | `warning` |
| image-off | `hide_image` |
| copy | `content_copy` |
| download | `download` |

> Nota de alcance: `app-icon` también se usa en `public-menu.component.ts` y `checkout/*-step.component.ts`
> (fuera de alcance). Esta tabla solo aplica a los usos de estos nombres **dentro** del panel de
> administración (ver research.md D5 para la lista exacta de módulos en alcance); los usos en el
> flujo público siguen renderizándose con `app-icon` sin cambios.

## Catálogo — reemplazos de emoji usados como ícono (por módulo, dentro de alcance)

| Módulo/archivo | Emoji actual | Significado | Ícono Material Icons (Outlined) |
|---|---|---|---|
| `sidebar.component.ts:41` | 🍦 (tenant por defecto) | Marca genérica de negocio | `storefront` |
| `sidebar.component.ts:41` | 🛡️ (super-admin) | Rol de super-administrador | `admin_panel_settings` |
| `admin-dashboard.component.ts:50,179` | 🍦 | Tarjeta/acceso "Productos" | `shopping_bag` |
| `admin-dashboard.component.ts` | 👥 | Tarjeta "Usuarios" | `group` |
| `admin-dashboard.component.ts` | 📋 | Tarjeta "Órdenes" | `receipt_long` |
| `admin-dashboard.component.ts` | 💰 | Tarjeta "Caja" | `payments` |
| `admin-dashboard.component.ts` | 🍽️ | Acceso "Terminal de mesas" | `restaurant` |
| `admin-dashboard.component.ts` | 🧾 | Acceso "Ventas" | `receipt` |
| `admin-dashboard.component.ts` | 🪑 | Acceso "Mesas" | `table_restaurant` |
| `cash-movement-modal.component.ts`, `cash-dashboard.component.ts:164`, `cash-arqueo-modal.component.ts` | ✕ | Cerrar modal | `close` |
| `inventory-page.component.ts:345` | ✕ | Cerrar modal | `close` |
| `users-page.component.ts:84` | 👥 | Estado vacío de usuarios | `group` |
| `user-role-modal.component.ts:29` | ✕ | Cerrar modal | `close` |
| `tenant-info.component.ts` | 🎨 | Sección de marca/branding | `palette` |
| `tenant-info.component.ts:61,170` | 🏪 | Sección de negocio | `storefront` |
| `tenant-info.component.ts` | 🧾 | Recibo/imprimir prueba | `receipt` |
| `tenant-info.component.ts` | 🖨️ | Impresora | `print` |
| `tenant-info.component.ts` | ✓ | Confirmación de guardado | `check` |
| `tenant-info.component.ts` | 👤 | Usuario | `person` |
| `tables-page.component.ts:55,80` | 🪑 | Mesa | `table_restaurant` |
| `tables-page.component.ts` | 📷 | Acción de foto/QR | `photo_camera` |
| `tables-page.component.ts` | ✏️ | Acción de editar | `edit` |
| `tables-page.component.ts` | 🔴 | Estado ocupada | `circle` (clase de color rojo ya existente) |
| `tables-page.component.ts` | 🟢 | Estado libre | `circle` (clase de color verde ya existente) |
| `table-sessions.component.ts:282` | ✅ | Confirmación de éxito | `check_circle` |
| `tables-page.component.ts` / `pos-checkout-panel.component.ts` | 🧾 | Imprimir cuenta/factura | `receipt` |
| `pos-order-panel.component.ts` | 🍽️ | Pedido para consumo en mesa | `restaurant` |
| `pos-order-panel.component.ts` | 📍 | Dirección | `location_on` |
| `pos-order-panel.component.ts` | 📞 | Teléfono de contacto | `call` |
| `pos-order-panel.component.ts` | 🛵 | Domicilio | `delivery_dining` |
| `pos-order-panel.component.ts` | ✓ | Confirmación | `check` |
| `pos-checkout-panel.component.ts` | ✏️ | Editar | `edit` |
| `pos-checkout-panel.component.ts` | 🔓 | Desbloquear/reabrir | `lock_open` |

## Catálogo — componentes compartidos con el menú público (dentro de alcance por research.md D6)

| Archivo | Emoji actual | Significado | Ícono Material Icons (Outlined) |
|---|---|---|---|
| `cart.component.ts` | 🛒 | Carrito | `shopping_cart` |
| `cart.component.ts` / modales compartidos | ✕ | Cerrar | `close` |
| `payment-attempt-review-panel.component.ts` | 💳 | Método de pago | `credit_card` |
| `pos-catalog-drawer.component.ts` / `product-select.component.ts` | 🏷️ | Etiqueta de precio/promoción | `sell` |

## Catálogo — SVG artesanales descubiertos durante la implementación (dentro de alcance)

Durante `/speckit-implement` se descubrió que el patrón "ícono SVG artesanal" no se limita al
componente `app-icon`: 14 archivos del panel de administración tienen íconos `<svg>` escritos a mano
directamente en su plantilla, sin pasar por `app-icon` ni por un emoji. El pedido original ("no puede
quedar ningún ícono generado por IA en formato SVG") y SC-002 ("cero íconos SVG artesanales
visibles") cubren este caso igual que a `app-icon` — se migran también.

| Archivo | Ícono SVG actual (forma) | Significado | Ícono Material Icons (Outlined) |
|---|---|---|---|
| `inventory-item-form.component.ts` | X (cerrar) | Cerrar modal | `close` |
| `cash-arqueo-modal.component.ts` | X (cerrar) | Cerrar modal | `close` |
| `stock-adjust-modal.component.ts` | X (cerrar) | Cerrar modal | `close` |
| `purchase-form.component.ts` | X (cerrar, cabecera) | Cerrar modal | `close` |
| `purchase-form.component.ts` | X (eliminar fila) | Quitar renglón de compra | `close` |
| `inventory-page.component.ts` | `+` | "Nuevo insumo" / "Nueva compra" | `add` |
| `cash-page.component.ts` | Reloj | Duración del turno | `schedule` |
| `cash-report.component.ts` | Candado cerrado | "Turno cerrado" | `lock` |
| `bill-summary.component.ts` | Moto de reparto | Ícono de domicilio | `delivery_dining` |
| `pos-tables-panel.component.ts` | Lupa | Buscador de mesas | `search` |
| `table-sessions.component.ts` | Círculo + cruz | Botón "nueva mesa" | `add_circle` |
| `header.component.ts` | 3 líneas horizontales | Abrir menú lateral | `menu` |
| `header.component.ts` | Campana | Notificaciones | `notifications` |
| `header.component.ts` | Flecha hacia abajo (rota) | Desplegar menú de usuario | `expand_more` |
| `header.component.ts` | Capas (igual a `app-icon` "layers") | "Mi plan" | `layers` |
| `header.component.ts` | Engranaje | "Cambiar contraseña" | `settings` |
| `header.component.ts` | Flecha saliendo de puerta | "Cerrar sesión" | `logout` |
| `pos-terminal-header.component.ts` | Dos personas (igual a `app-icon` "users") | Turno/cajero activo | `group` |
| `pos-terminal-header.component.ts` | Tarjeta (rect+línea) | "Abrir turno de caja" | `credit_card` |
| `pos-terminal-header.component.ts` | Candado cerrado | "Bloquear terminal" | `lock` |
| `manual-order-page.component.ts` | Flecha izquierda | "Volver a la Terminal" | `arrow_back` |
| `manual-order-page.component.ts` | Lupa | Buscador de productos | `search` |
| `manual-order-page.component.ts` | Esquinas de escaneo | "Escanear código" | `qr_code_scanner` |
| `manual-order-page.component.ts` | Mesa (badge) | Tipo de orden: mesa | `table_restaurant` |
| `manual-order-page.component.ts` | Bolsa (badge) | Tipo de orden: para llevar | `shopping_bag` |
| `manual-order-page.component.ts` | Moto de reparto (badge) | Tipo de orden: domicilio | `delivery_dining` |
| `manual-order-page.component.ts` | Carrito (avatar de tarjeta de producto) | Agregar producto | `shopping_cart` |
| `manual-order-page.component.ts` | Mesa (pestaña) | Pestaña "Mesa" | `table_restaurant` |
| `manual-order-page.component.ts` | Bolsa (pestaña) | Pestaña "Para llevar" | `shopping_bag` |
| `manual-order-page.component.ts` | Moto (pestaña) | Pestaña "Domicilio" | `delivery_dining` |
| `manual-order-page.component.ts` | Lápiz (x2, editar nombre) | "Editar nombre" | `edit` |
| `manual-order-page.component.ts` | Persona | Campo "Cliente / Para llevar" | `person` |
| `manual-order-page.component.ts` | Lápiz (x2, notas) | "Ver/Editar Notas" | `edit` |
| `manual-order-page.component.ts` | Cono/copa (x2) | "Modificar Toppings" — **forma de cono de helado, ver nota de heladería abajo** | `tune` |
| `manual-order-page.component.ts` | Caneca (x2) | "Eliminar ítem" | `delete` |
| `manual-order-page.component.ts` | Tarjeta (botón de enviar) | Botón de confirmar orden | `credit_card` |

> **Nota de heladería**: el ícono de "Modificar Toppings" en `manual-order-page.component.ts` tiene
> literalmente forma de cono/copa de helado (path Lucide `ice-cream`) — un tercer punto con temática
> de heladería no detectado durante `/speckit-plan` (que solo buscó emoji 🍦, no formas de SVG). Se
> reemplaza por `tune` (mismo ícono neutro que ya usa `option-groups`), consistente con la Historia 2
> de spec.md, aunque técnicamente se descubrió durante la implementación de la Historia 1 (ambas son
> P1, mismo criterio de aceptación).

## Fuera de catálogo (no se migra)

- Emoji dentro de texto/copy de mensajes (p. ej. saludo con 👋 en el dashboard) — contenido, no
  ícono independiente (ver spec.md, Edge Cases).
- 🎁 en `pos-terminal.store.ts` — es una etiqueta de datos para nombrar combos, no un ícono
  renderizado en una plantilla; no aplica FR-004.
- Cualquier glifo dentro de `modules/tables/pages/{public-menu, diner-shell, expired-qr,
  checkout/*}` y `modules/super-admin/**` — fuera de alcance (research.md D5), salvo los
  componentes compartidos ya listados arriba.
