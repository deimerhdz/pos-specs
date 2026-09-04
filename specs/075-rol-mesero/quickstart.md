# Quickstart de validación: Rol Mesero

**Spec**: [spec.md](./spec.md) | **Contratos**: [contracts/](./contracts/)

Guía para verificar manualmente (o como base de tests de integración) que el
rol Mesero funciona de punta a punta, en ambos repos.

## Prerrequisitos

- `pos-backend` corriendo localmente con la migración de esta funcionalidad
  aplicada (`alembic upgrade head`) — confirma que `shared.roles` tiene una
  fila `MESERO`.
- `pos-heladeria` corriendo localmente (`npm start`) apuntando a ese backend.
- Un tenant existente con al menos una mesa configurada y un producto activo
  en el menú (para poder tomar un pedido de prueba).
- Un usuario Admin del tenant, para asignar el rol.

## 1. Asignar el rol Mesero (User Story 1)

1. Iniciar sesión como Admin del tenant.
2. Ir a Usuarios → Agregar usuario → seleccionar rol **Mesero** → invitar con
   un correo de prueba.
3. Aceptar la invitación (o, más rápido para pruebas locales, cambiar el rol
   de un usuario Cajero ya existente a **Mesero** desde el listado de
   usuarios).
4. **Verificar**: el usuario aparece en el listado etiquetado como "Mesero"
   (no como un código técnico).

## 2. Navegación restringida (User Story 2)

1. Iniciar sesión con el usuario Mesero.
2. **Verificar**: el destino tras el login es `/dashboard/mesas-sesiones`
   (Terminal de Mesas), no el Dashboard administrativo.
3. **Verificar**: el menú de navegación solo muestra "Terminal de mesas" y
   "Órdenes" (además de "Mi cuenta"/"Mi plan").
4. Desde la Terminal de Mesas: abrir una mesa, tomar un pedido, enviarlo a
   cocina, y cobrar la cuenta. **Verificar**: cada paso funciona sin errores,
   igual que con un usuario Cajero.
5. Ir a Órdenes: **verificar** que se puede consultar el listado y el
   detalle de una orden, y que no aparece ninguna acción que modifique su
   estado.
6. Con la sesión de Mesero activa, navegar por URL directa a
   `/dashboard/inventario` (o `/dashboard/reports`, `/dashboard/users`,
   `/dashboard/caja`, `/dashboard/ajustes`). **Verificar**: redirige de
   inmediato a `/dashboard/mesas-sesiones`, sin mostrar la pantalla ni un
   error técnico.

## 3. Bloqueo real del lado del servidor (User Story 3)

Con el token JWT del usuario Mesero (capturarlo del `localStorage` tras el
login, o del panel de red del navegador):

1. Llamar directamente, con ese token, a un endpoint fuera de su alcance —
   por ejemplo `GET /inventario`, `GET /reports`, `GET /cash/registers`,
   `POST /sales`, `GET /users`. **Verificar**: cada uno responde `403`, no
   `200` ni `404`.
2. Llamar a un endpoint dentro de su alcance — por ejemplo `GET /orders`,
   `GET /table-sessions`, `POST /orders/{id}/pay` sobre una orden de prueba.
   **Verificar**: responde con normalidad (mismo código que obtendría un
   Cajero).
3. Repetir el paso 1 con el token de un usuario **Cajero** o **Admin** sobre
   los mismos endpoints que antes fueron bloqueados para Mesero.
   **Verificar**: el comportamiento es idéntico al que tenían antes de esta
   funcionalidad (sin ninguna restricción nueva para esos dos roles —
   SC-004).

## 4. Edge cases

- Cambiar el rol de un usuario Cajero con sesión activa a Mesero (o
  viceversa) sin que cierre sesión. **Verificar**: en la siguiente solicitud
  (recargar la página, o la siguiente llamada a la API), el nuevo alcance ya
  aplica — no requiere volver a iniciar sesión.
- Con un tenant que no tiene ningún usuario Mesero, verificar que el resto
  del sistema (Admin, Cajero) funciona exactamente igual que antes de esta
  funcionalidad.

## Referencias

- Matriz completa de endpoints permitidos/bloqueados:
  [contracts/backend-endpoint-access.md](./contracts/backend-endpoint-access.md)
- Matriz completa de rutas/navegación del frontend:
  [contracts/frontend-route-access.md](./contracts/frontend-route-access.md)
- Decisiones técnicas y su justificación: [research.md](./research.md)
