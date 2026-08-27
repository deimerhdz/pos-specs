# Contratos — Corrección de bugs y mejoras — Menú QR

Esta spec **no agrega, ni modifica, ni retira ningún endpoint** de `pos-backend` (FR-007, FR-012,
FR-026; ver también `plan.md` §Technical Context). Los cuatro bugs se resuelven enteramente dentro
de `pos-heladeria`. Este documento existe por trazabilidad (Principio XII de la Constitución): deja
constancia de los contratos existentes de los que cada corrección depende, sin cambiarlos.

## Bug 1 — Invalidación de acceso

| Endpoint | Uso en esta spec | Cambio de contrato |
|---|---|---|
| `POST /cart/sessions` (`open_session`, `pos-backend/app/api/v1/cart/service.py:109-166`) | Crea/une la sesión del comensal al escanear el QR o introducir el nombre. La corrección de Bug 1 sigue llamándolo exactamente igual, solo que ahora el frontend no ofrece el camino hacia esta llamada cuando la pestaña tiene la marca de acceso cerrado (ver `data-model.md`). | Ninguno |
| `POST /cart/leave` (`leave_session`, `pos-backend/app/api/v1/cart/service.py:507-515`) | Invalida `SessionParticipant.status` al cerrar sesión (best-effort, ya protegido por FR-001). | Ninguno |
| `GET /menu/qr-token/{token}` (resolución del token QR de la mesa, `menu/router.py`) | Resuelve mesa/menú a partir del token de la URL en cada `ngOnInit`; sigue siendo stateless y sin cambios — la marca de acceso cerrado se evalúa **después** de esta llamada, del lado del cliente. | Ninguno |

## Bug 2 — Descarga de QR de mesas

| Endpoint | Uso en esta spec | Cambio de contrato |
|---|---|---|
| `GET /orders/tables/{table_id}/qr-token` (`pos-backend/app/api/v1/orders/router.py:148-166`, `mint_qr_token`) | Emite el token firmado que ambas variantes (Mostrador/Sticker) codifican en el QR. | Ninguno — mismo token, mismo destino codificado (FR-012). |

No hay ningún endpoint de "descarga de PNG" en el backend hoy (la composición del PNG, incluida la
nueva composición con el identificador de mesa, ocurre enteramente en el navegador — ver
`research.md` Decisión 3) ni esta spec agrega uno.

## Bug 3 — Placeholder del catálogo

Sin ningún endpoint involucrado — el catálogo ya llega con `image_url` nulo o no (`GET
/menu/qr-token/{token}`, sin cambios); la corrección es puramente de renderizado en
`public-menu.component.ts`.

## Bug 4 — Botón "Copiar insumos"

Sin ningún endpoint propio hoy (FR-026) ni esta spec agrega uno — la operación de copiar solo
modifica el `draft` en memoria del formulario; se persiste al guardar el producto completo con los
endpoints ya existentes de creación/edición de producto y de receta/insumos, sin ningún cambio de
contrato.
