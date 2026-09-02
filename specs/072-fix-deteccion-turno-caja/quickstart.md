# Quickstart — validación de la spec 072

**Spec**: [spec.md](./spec.md) · **Plan**: [plan.md](./plan.md) · **Fecha**: 2026-09-02

Guía de validación ejecutable: qué correr y qué esperar para dar la corrección por completa
(Principio X). No contiene implementación — el detalle normativo está en
[contracts/](./contracts/) y las tareas en `tasks.md`.

---

## 1. Preparar el entorno

```bash
# Backend (sin cambios en esta spec, pero necesario para levantar la API)
cd ../pos-backend
docker compose up -d postgres redis
uvicorn app.main:app --reload          # http://localhost:8000

# Frontend
cd ../pos-heladeria
npm start                              # http://localhost:4200
```

## 2. Batería automatizada

```bash
# Backend — no debería verse afectado; correr igual como red de seguridad
cd ../pos-backend
python -m unittest discover -s app/characterization_tests -p 'test_*.py' -v

# Frontend
cd ../pos-heladeria
npm test
```

**Criterio de aceptación**: 0 fallos nuevos. En particular, los 35 tests de
`pos-terminal.store.spec.ts` que hacen `store.orders.set([...])` directamente deben seguir en
verde sin ninguna petición HTTP nueva inesperada (research.md D1) — si alguno empieza a fallar por
un `http.verify()` con peticiones sin consumir, es señal de que el disparador quedó enganchado a
un `effect()` reactivo en vez de dentro de `reloadOrders()`.

## 3. Validación manual por historia

Usar dos pestañas/dispositivos: uno como "Terminal" (el que reproduce el bug) y otro para simular
"otro cajero/dispositivo" que abre el turno.

### US1 — Cobrar el pedido que llega a una mesa ya seleccionada

1. Limpiar `localStorage` del navegador de la Terminal (`localStorage.removeItem('cash.register')`
   en la consola, o modo incógnito nuevo).
2. Desde otra sesión/dispositivo, abrir un turno de caja (o usar uno ya abierto por otro usuario).
3. En la Terminal, seleccionar una mesa libre. **No** tocarla de nuevo.
4. Desde el menú QR (u otra pestaña simulando al comensal), enviar un pedido a esa mesa.
5. **Esperado**: en menos de ~2 segundos desde que aparece "Pagos por confirmar", el botón
   "Confirmar efectivo" está habilitado — sin mensaje de "Abre un turno de caja...".
6. Repetir seleccionando activamente una mesa que ya tenía pedido (en vez de esperar a que
   llegue) — también debe funcionar sin pasos previos (US1, escenario 3).

### US2 — El bloqueo se mantiene sin turno abierto

1. Sin ningún turno de caja abierto en el tenant, repetir el mismo flujo.
2. **Esperado**: "Confirmar efectivo" sigue bloqueado con el mensaje de siempre.
3. Con un turno abierto y ya detectado, cerrarlo. Verificar que el siguiente pedido nuevo ya no
   se puede cobrar sin abrir un turno nuevo.

### US3 — Dos turnos abiertos a la vez

1. Abrir turno en dos cajas distintas del mismo tenant.
2. En un navegador de Terminal sin ninguna operada explícitamente, repetir el flujo de un pedido
   nuevo.
3. **Esperado**: sigue pidiendo "Operar" una caja desde el módulo de Caja — no cobra
   automáticamente contra ninguna de las dos.

## 4. Regresión explícita (lo que no debe cambiar)

- Con solo una mesa libre seleccionada (sin ningún pedido en ningún lado), la Terminal sigue sin
  pedir métodos de pago ni turno de caja (spec 059 FR-001) — verificable en la pestaña Red del
  navegador.
- Ningún endpoint de `pos-backend` cambia; `GET /cash/registers` y
  `GET /cash/shifts/current?cash_register_id=` siguen respondiendo igual.
- Una vez resuelto el turno en una sesión de pantalla, no se repite la petición en cada sondeo
  (spec 059 FR-003) — verificable viendo que solo hay una llamada a `/cash/shifts/current` por
  caja mientras el turno siga abierto.
