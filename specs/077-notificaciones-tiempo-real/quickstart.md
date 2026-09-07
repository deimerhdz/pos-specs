# Quickstart: validar Notificaciones en Tiempo Real Multi-Tenant

**Spec**: [spec.md](spec.md) | **Contratos**: [contracts/](contracts/)

Guía para comprobar, de punta a punta, que la funcionalidad cumple el checklist de aceptación de la spec. No repite código de implementación — solo comandos y resultados esperados.

## Prerrequisitos

- Redis corriendo (ya lo requiere `/realtime` hoy) y `REALTIME_ENABLED=true`.
- Backend: `cd ../pos-backend && uvicorn app.main:app --reload` (Alembic ya migrado con la revisión de esta spec — ver `data-model.md`).
- Frontend: `cd ../pos-heladeria && ng serve`.
- Dos usuarios de prueba: un cajero del tenant A (login normal del POS) y un enlace de menú QR de una mesa de ese mismo tenant A (URL con el token del comensal).
- Para el escenario multi-tenant: un segundo tenant B con su propio cajero.
- Navegador con soporte de Push API (Chrome/Edge/Firefox) para los pasos de push.

## 1. El cajero ve el pedido nuevo sin estar en Terminal de Mesas (US1, SC-001/SC-002)

1. Iniciar sesión como cajero del tenant A y quedarse en la sección **Ventas**.
2. En otra pestaña/dispositivo, abrir el menú QR de una mesa de ese tenant y confirmar un pedido.
3. **Esperado**: en menos de 5 s, en la sección Ventas aparece un aviso visual (campanita/contador) y se escucha el sonido — sin haber navegado a Terminal de Mesas.
4. Verificar en la pestaña de red del navegador que la conexión `GET /realtime/stream` sigue abierta aunque se navegue entre secciones del POS (antes se cerraba al salir de Terminal de Mesas).

## 2. Push del navegador con la pestaña sin foco (US3, SC-003)

1. Con el cajero del paso 1, conceder el permiso de notificaciones push cuando el POS lo solicite (primer login o desde el centro de notificaciones).
2. Minimizar el navegador o cambiar a otra pestaña.
3. Repetir el paso 2 del escenario 1 (confirmar otro pedido).
4. **Esperado**: llega una notificación del sistema operativo/navegador equivalente al aviso en la aplicación.
5. Repetir con la pestaña del POS enfocada: **esperado** no llega push duplicado (el Service Worker detecta la pestaña enfocada vía `clients.matchAll()` y omite `showNotification()` — research.md §5).

## 3. El comensal ve la confirmación de pago en tiempo real (US4, SC-004)

1. Con el pedido del paso 1 ya creado, cobrar la cuenta desde el POS del cajero (Terminal de Mesas o panel de cobro).
2. **Esperado**: la pestaña del menú QR del comensal (sin recargar) muestra la confirmación de pago en menos de 5 s.

## 4. Aislamiento entre tenants (US2, SC-005)

Manual:
1. Login simultáneo de un cajero del tenant A y uno del tenant B.
2. Confirmar un pedido en una mesa del tenant A.
3. **Esperado**: el cajero del tenant B no ve ningún aviso ni cambio de contador.

Automatizado (obligatorio para el checklist de aceptación de la spec):

```bash
cd ../pos-backend
python -m unittest app.characterization_tests.test_notifications -v
```

**Esperado**: el caso de aislamiento entre tenants pasa en verde — ver `contracts/realtime-events.md` y `research.md` §8 para el diseño de la prueba.

## 5. Recuperación tras desconexión (US5, SC-006)

1. Con el cajero del paso 1 conectado, simular pérdida de red (DevTools → Network → Offline) o cerrar la pestaña del POS.
2. Confirmar un par de pedidos nuevos para el tenant A mientras está desconectado.
3. Reconectar (Network → Online, o reabrir el POS y volver a iniciar sesión si la pestaña se cerró).
4. **Esperado**: el centro de notificaciones muestra las notificaciones generadas durante la desconexión, en orden, sin duplicados. Verificable también directamente:

```bash
curl -H "Authorization: Bearer <token-del-cajero>" \
  "http://localhost:8000/api/v1/notifications?after_id=<id-de-la-ultima-notificacion-vista>"
```

## 6. Canal nuevo sin tocar el código de negocio (SC-007)

Verificación de diseño, no de UI: confirmar que `app/core/notifications/dispatch.py` (`notify_order_created`, `notify_payment_completed`) es el único punto que conoce la lista de canales habilitados, y que ninguno de los puntos de llamada de negocio (`cart/router.py`, `orders/router.py`, `table_sessions/service.py`) referencia una clase de canal directamente — ver `research.md` §4.

## 7. Purga por retención (SC-008)

```bash
cd ../pos-backend
python -c "from app.core.notifications.purge import purge_expired_notifications; print(purge_expired_notifications())"
```

1. Insertar (o adelantar `purge_at` de) una `NotificationEvent` de prueba a una fecha pasada.
2. Ejecutar el comando anterior.
3. **Esperado**: la fila purgada ya no aparece en `GET /notifications` ni en una consulta directa a la tabla.
4. Confirmar que el valor de retención se lee de `tenants.notification_retention_days` (90 por defecto) y no de una constante en código — cambiar la columna de un tenant de prueba a `1` día y repetir con una notificación de ayer confirma que es configurable por tenant.

## 8. Multi-instancia sin pérdida ni duplicado (SC-009)

Ya cubierto por el diseño del bus existente (research.md §1); no requiere una prueba nueva de esta spec más allá de correr dos procesos `uvicorn` (o `--workers 2`) contra el mismo Redis y repetir el escenario 1, confirmando que el aviso llega exactamente una vez sin importar a qué proceso esté conectado el cajero.
