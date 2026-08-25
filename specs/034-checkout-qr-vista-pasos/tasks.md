---

description: "Task list template for feature implementation"
---

# Tasks: Vista de Pasos para Revisión y Pago del Menú QR

**Input**: Documentos de diseño de `/specs/034-checkout-qr-vista-pasos/`

**Prerequisitos**: plan.md, spec.md, research.md, data-model.md, contracts/, quickstart.md

**Tests**: no se solicitaron tests explícitamente en el spec ni el plan — esta lista no incluye
tareas de test dedicadas; la verificación se hace con `quickstart.md` (Fase Final) y con el script
autoejecutable de backend ya requerido por el Principio X de la constitución.

**Organización**: las tareas se agrupan por historia de usuario (spec 034, Historias 1-4) para que
cada una pueda implementarse y verificarse de forma independiente, según el Principio VI
(Evolución Incremental) marcado como punto de atención en `plan.md`.

## Formato: `[ID] [P?] [Story] Descripción`

- **[P]**: puede ejecutarse en paralelo (archivo distinto, sin dependencias pendientes)
- **[Story]**: historia de usuario a la que pertenece (US1-US4)
- Cada tarea incluye la ruta de archivo exacta

## Convenciones de ruta

- Backend: `/home/deimer/Documents/projects/inventario/pos-backend/app/api/v1/cart/`
- Frontend: `/home/deimer/Documents/projects/inventario/pos-heladeria/src/app/`

---

## Fase 1: Setup

**Propósito**: crear el andamiaje de archivos y rutas donde vivirá la nueva vista.

- [X] T001 Crear la carpeta `pos-heladeria/src/app/modules/tables/pages/checkout/` con archivos
  vacíos para los cuatro componentes de paso (`review-step.component.ts`,
  `payment-method-step.component.ts`, `transfer-details-step.component.ts`,
  `confirmation-step.component.ts`) y el store de progreso (`checkout-progress.store.ts`), según la
  estructura de `plan.md`.
- [X] T002 Registrar la ruta hija `menu/t/:token/checkout` (con subrutas para cada paso) en
  `pos-heladeria/src/app/app.routes.ts`, apuntando a los componentes creados en T001, con carga
  perezosa (mismo patrón ya usado por `PublicMenuComponent`, `app.routes.ts:63-69`). (depende de T001)

---

## Fase 2: Foundational (bloqueante para todas las historias)

**Propósito**: dejar la navegación básica entre pasos funcionando y retirar el modal actual, para
que ninguna historia tenga que convivir con las dos implementaciones a la vez.

**⚠️ CRÍTICO**: ninguna historia de usuario puede empezar a verificarse hasta completar esta fase.

- [X] T003 Implementar la navegación básica entre los cuatro pasos (revisión → método →
  transferencia → confirmación) sin persistencia todavía, en los componentes de
  `pos-heladeria/src/app/modules/tables/pages/checkout/` creados en T001. (depende de T001, T002)
- [X] T004 Retirar el modal de revisión/pago actual (signals `reviewOpen`, `reviewStep`,
  `reviewMethod`, `reviewReceiptUrl`, `reviewSubmitting`, `reviewError` y su plantilla,
  `pos-heladeria/src/app/modules/tables/pages/public-menu.component.ts`), y hacer que el botón
  "Enviar pedido" navegue a `menu/t/:token/checkout` en lugar de abrir el modal. (depende de T003)

**Checkpoint**: la vista de pasos existe y es navegable (sin persistencia, sin datos de pago con
imagen, sin iconos nuevos); a partir de aquí cada historia agrega una capa independiente.

---

## Fase 3: Historia de Usuario 1 - El comensal retoma el pago exactamente donde iba si recarga la página (Priority: P1) 🎯 MVP

**Objetivo**: que recargar la página en cualquier paso, antes de que el pedido exista, devuelva al
comensal exactamente a donde iba, sin repetir selecciones ya hechas.

**Independent Test**: avanzar hasta elegir un método de transferencia, recargar la página, y
verificar que se vuelve al mismo paso con el método ya elegido; luego subir un comprobante y
recargar de nuevo, verificando que no se pide subirlo otra vez y que solo se crea un pedido.

### Implementación de la Historia 1

- [X] T005 [P] [US1] Implementar `checkout-progress.store.ts`
  (`pos-heladeria/src/app/modules/tables/pages/checkout/checkout-progress.store.ts`): lectura,
  escritura y borrado de `localStorage` con clave por `table_session_id`/`participant_id`
  (`step`, `payment_method_id`, `receipt_file_url`, `saved_at`), y una función que considera el
  registro inválido si `saved_at` cae fuera de la ventana de sesión vigente (FR-009, ver
  `data-model.md`).
- [X] T006 [P] [US1] Al confirmar un método de pago en `payment-method-step.component.ts`, guardar
  `step` y `payment_method_id` vía el store de T005 (FR-005). (depende de T005)
- [X] T007 [P] [US1] En `transfer-details-step.component.ts`, al completar con éxito la subida del
  comprobante (presign + `PUT`), guardar `receipt_file_url` en el store de T005 **antes** de llamar
  `POST /cart/submit` (FR-006). (depende de T005)
- [X] T008 [US1] Al inicializar cualquier componente de paso en `checkout/`, hidratar desde el store:
  si hay `receipt_file_url` guardado, saltar directo a reintentar `POST /cart/submit` sin pedir el
  archivo de nuevo; si solo hay `payment_method_id`, mostrar el paso de transferencia con ese método
  ya elegido (FR-005, FR-007). (depende de T006, T007)
- [X] T009 [US1] Si `POST /cart/submit` ya se completó con éxito para este comensal (pedido
  existente), redirigir automáticamente a la pantalla de confirmación en vez de mostrar la vista de
  revisión (FR-008). (depende de T008)
- [X] T010 [US1] Al hidratar, verificar el `payment_method_id` guardado contra
  `GET /cart/payment-methods` vigente; si ya no está activo, limpiar solo ese campo del progreso y
  pedir elegir de nuevo, conservando el resto (resumen del pedido) (FR-010). (depende de T008)
- [X] T011 [US1] Limpiar el registro completo del store en cuanto `POST /cart/submit` responde con
  éxito — a partir de ahí aplica **Orden**/**Intento de Pago**/**Comprobante** (spec 024/025), no
  este progreso. (depende de T008)

**Checkpoint**: recargar la página en cualquier paso, antes de que el pedido exista, ya no hace
perder el progreso — Historia 1 verificable de forma independiente.

---

## Fase 4: Historia de Usuario 2 - El comensal navega la revisión y el pago en una vista propia, con pasos claros (Priority: P1)

**Objetivo**: que el indicador de paso, la navegación hacia atrás y la salida sin crear pedido sean
explícitos y visibles en la nueva vista (más allá de la navegación básica ya cableada en la Fase 2).

**Independent Test**: abrir la revisión desde el carrito y verificar el indicador de paso visible;
volver de "datos de transferencia" a "selección de método" y elegir otro método sin rastro del
anterior; salir sin completar el pago y verificar que el carrito no cambia y no se crea ningún
pedido.

### Implementación de la Historia 2

- [X] T012 [US2] Agregar un indicador visible de paso (ej. "Paso 2 de 3: Método de pago") en el
  contenedor común de `pos-heladeria/src/app/modules/tables/pages/checkout/`, reflejando el paso
  actual (FR-002).
- [X] T013 [US2] Implementar la navegación "volver" entre pasos (de datos de transferencia a
  selección de método), permitiendo elegir uno distinto (efectivo u otra transferencia) sin
  restricción mientras el pedido no exista (FR-003). (depende de T012)
- [X] T014 [US2] Implementar la salida de la vista (botón/navegación fuera) en cualquier paso previo
  a que el pedido exista, sin crear ningún pedido y preservando el carrito exactamente igual
  (FR-004).
- [X] T015 [US2] Al volver y elegir un método de transferencia distinto al guardado, limpiar del
  store (T005) cualquier rastro del método anterior (`payment_method_id`, `receipt_file_url`) para
  que no quede mezclado con el nuevo. (depende de T005, T013)

**Checkpoint**: la vista se siente y se navega como una pantalla propia con pasos claros — Historia
2 verificable de forma independiente.

---

## Fase 5: Historia de Usuario 3 - El comensal ve el número de cuenta y el código QR de pago con claridad (Priority: P2)

**Objetivo**: que el código QR de pago se muestre como imagen y el número de cuenta como texto
legible, en vez de texto/URL plano.

**Independent Test**: elegir un método de transferencia con número de cuenta e imagen de QR
configurados, y verificar que el número se lee como texto y el QR se ve como imagen; elegir uno sin
imagen de QR y verificar que no queda ningún espacio roto.

### Implementación de la Historia 3

- [X] T016 [P] [US3] Agregar el campo `fields: list[PaymentMethodFieldDefinition]` a
  `DinerPaymentMethod` en `pos-backend/app/api/v1/cart/schemas.py` (mismo tipo que ya usa
  `sales/schemas.py:57` / `super_admin/schemas.py:18-26`). Ver `contracts/cart-payment-methods.md`.
- [X] T017 [US3] Modificar `list_payment_methods` en `pos-backend/app/api/v1/cart/service.py:648-655`
  para incluir `fields`, tomado de `PaymentMethodCatalog.fields`
  (`models/payment_method_catalog.py:23-26`), en la respuesta de `GET /cart/payment-methods`.
  (depende de T016)
- [X] T018 [P] [US3] Actualizar la interfaz TypeScript del método de pago del comensal
  (`pos-heladeria/src/app/.../diner.interface.ts`, tipo `DinerPaymentMethod`) para incluir `fields`.
- [X] T019 [US3] En `transfer-details-step.component.ts`, renderizar cada clave de `payment_info`
  según su `format` en `fields`: `image` → `<img>`; `text`/`numeric` → texto legible y etiquetado
  (FR-011, FR-012). (depende de T018)
- [X] T020 [US3] Manejar el caso sin imagen de QR configurada: no renderizar ningún `<img>` ni ícono
  de "imagen no encontrada" cuando ningún campo del método tenga `format: image` (FR-013). (depende
  de T019)
- [X] T021 [US3] Escribir un script autoejecutable de backend (nuevo o ampliando uno existente bajo
  `pos-backend/app/scripts/`) que verifique que `GET /cart/payment-methods` incluye `fields` con el
  `format` correcto por cada campo (Principio X de la constitución). (depende de T017)

**Checkpoint**: los datos de pago de transferencia se ven con claridad — Historia 3 verificable de
forma independiente.

---

## Fase 6: Historia de Usuario 4 - Los iconos de la pantalla de pago se ven profesionales y consistentes (Priority: P3)

**Objetivo**: que ningún icono de esta vista sea un emoji; todos vienen del sistema de iconos ya
existente en la aplicación.

**Independent Test**: recorrer los cuatro pasos y verificar que ningún icono es un emoji — todos son
elementos generados por `app-icon`.

### Implementación de la Historia 4

- [X] T022 [P] [US4] Agregar los `@case` faltantes en
  `pos-heladeria/src/app/shared/icon/icon.component.ts` para los iconos de este flujo (transferencia,
  adjuntar comprobante, quitar comprobante, volver, confirmar), reutilizando los `@case` `cash` /
  `receipt` ya existentes donde corresponda.
- [X] T023 [US4] Reemplazar los emoji del flujo de checkout (💵, 📲, 📤/📎, ✕, ←, hoy en
  `public-menu.component.ts:599` y equivalentes ya movidos a `checkout/`) por
  `<app-icon name="...">` en los cuatro componentes de paso. (depende de T022)

**Checkpoint**: todos los iconos de la vista son vectoriales y consistentes — Historia 4 verificable
de forma independiente.

---

## Fase Final: Pulido y verificación cruzada

**Propósito**: validar el conjunto y confirmar que no hay regresiones fuera del alcance de esta spec.

- [ ] T024 Ejecutar los 4 escenarios de `quickstart.md` completos contra un entorno local
  (`pos-backend` + `pos-heladeria` corriendo, tenant de prueba con métodos configurados con y sin
  imagen de QR). **Pendiente de verificación manual en navegador** — `ng build` de `pos-heladeria`
  compila limpio con la nueva vista y los 3 scripts de backend (`test_cart_payment_methods_fields`,
  suite `unittest` de spec 032) pasan, pero el click-through visual de los 4 escenarios no se
  ejecutó desde este agente (sin acceso de red al `fastapi dev`/`ng serve` que ya tenías corriendo).
- [X] T025 [P] Confirmar que el modal de reintento de pago tras rechazo de comprobante (spec 024,
  Historia 5, sobre un pedido ya creado) sigue intacto — no fue tocado por T001-T023. Verificado por
  inspección: `git diff` de `pos-heladeria` no toca `payingOrder`/`openPayment`/`closePayment`/
  `choosePaymentMethod`/`onReceiptSelected` ni su plantilla; el único cambio que le llega es aditivo
  (`fields` en `GET /cart/payment-methods`, que ese modal no lee).
- [X] T026 [P] Confirmar que la pantalla de cobro en caja (spec 032, FR-012a) sigue mostrando los
  métodos de pago solo por nombre, sin datos de integración, para el cajero. Verificado por
  inspección: ningún archivo de `dashboard/`/`sales/` (frontend) ni `super_admin/`/`sales/` (backend)
  fue tocado — el cambio de esta spec se limita a `cart/` (backend) y `modules/tables/` (frontend).

---

## Dependencias y orden de ejecución

### Dependencias entre fases

- **Setup (Fase 1)**: sin dependencias — puede empezar de inmediato.
- **Foundational (Fase 2)**: depende de Setup — bloquea las cuatro historias.
- **Historias de usuario (Fase 3-6)**: todas dependen de que Foundational esté completa.
  - **Historia 1 y Historia 2 comparten el mismo componente de contenedor/pasos** (ambas son P1 y
    están explícitamente acopladas en el spec: la Historia 2 es "el cambio de interfaz que hace
    posible la Historia 1"). Se recomienda implementarlas en secuencia (Fase 3 antes que Fase 4) por
    el mismo desarrollador o equipo, no en paralelo, para evitar conflictos de archivo en los
    componentes de paso.
  - **Historia 3** depende únicamente de Foundational — puede avanzar en paralelo a la Historia 1/2
    en el backend (T016-T017, T021), pero su parte de frontend (T019-T020) toca el mismo archivo
    (`transfer-details-step.component.ts`) que T007/T008 de la Historia 1 — coordinar si se trabaja
    en paralelo.
  - **Historia 4** depende únicamente de Foundational — su parte de frontend (T023) toca los cuatro
    componentes de paso, así que conviene hacerla al final para no generar conflictos de merge con
    las demás historias.
- **Pulido (Fase Final)**: depende de que todas las historias que se vayan a entregar estén
  completas.

### Dentro de cada historia

- El store de progreso (T005) antes que cualquier componente que lo use (T006-T011, T015).
- El campo `fields` del backend (T016-T017) antes que el frontend lo consuma (T018-T020).
- Los `@case` de iconos (T022) antes que su reemplazo en los componentes (T023).

### Oportunidades de paralelismo

- T005 puede iniciarse tan pronto termine Foundational; T006 y T007 pueden avanzar en paralelo una
  vez T005 esté listo (archivos distintos: paso de método vs. paso de transferencia).
- T016 y T018 pueden avanzar en paralelo (backend vs. frontend) antes de que exista integración real
  entre ambos.
- T022 (iconos) puede avanzar en paralelo a cualquier otra historia, ya que solo toca
  `icon.component.ts`.
- T025 y T026 (verificación de no-regresión) pueden ejecutarse en paralelo entre sí al final.

---

## Ejemplo de ejecución en paralelo: Historia 1

```bash
# Una vez completado T005 (store), lanzar juntos:
Task: "Guardar step/payment_method_id al confirmar método en payment-method-step.component.ts"
Task: "Guardar receipt_file_url al completar la subida en transfer-details-step.component.ts"
```

---

## Estrategia de implementación

### MVP (Historias 1 + 2 juntas)

A diferencia de la regla general de "MVP = solo la primera historia", aquí **ambas historias P1
(1 y 2) forman el MVP real**: una vista de pasos sin resiliencia ante recarga no resuelve el dolor
descrito por el usuario, y la resiliencia ante recarga no tiene dónde vivir sin la vista propia. El
plan (`plan.md`, Constitution Check, Principio VI) ya señala esta relación explícitamente.

1. Completar Fase 1: Setup
2. Completar Fase 2: Foundational (bloqueante)
3. Completar Fase 3: Historia 1
4. Completar Fase 4: Historia 2
5. **DETENER Y VALIDAR**: ejecutar los escenarios 1 y 2 de `quickstart.md`
6. Desplegar/demostrar el MVP si está listo

### Entrega incremental

1. Setup + Foundational → base lista
2. Historia 1 + Historia 2 → MVP (resiliencia + vista propia) → validar → desplegar
3. Historia 3 (datos de pago con imagen) → validar → desplegar
4. Historia 4 (iconos) → validar → desplegar
5. Fase Final (pulido y no-regresión) → cierre de la funcionalidad

---

## Notas

- [P] = archivos distintos, sin dependencias pendientes.
- Cada historia debe poder completarse y probarse de forma independiente, salvo la relación
  explícita Historia 1 ↔ Historia 2 documentada arriba.
- Ninguna tarea modifica el modal de reintento de pago tras rechazo (spec 024) ni la pantalla de
  cobro en caja (spec 032) — ver T025/T026.
- Sin tareas de test dedicadas (no solicitadas); la verificación es `quickstart.md` + el script
  autoejecutable de T021.
