# Quickstart — Corrección de bugs y mejoras — Menú QR

Guía de validación extremo a extremo de los cuatro bugs, una vez implementados. No incluye código
de implementación — solo pasos ejecutables y el resultado esperado. Ver `spec.md` para los criterios
de aceptación completos y `data-model.md`/`research.md` para el detalle técnico de cada mecanismo.

## Prerrequisitos

- `pos-backend` corriendo localmente (sin cambios de esta spec, pero necesario para servir el
  catálogo/sesión reales) y `pos-heladeria` corriendo (`ng serve` o equivalente) contra ese backend.
- Al menos dos mesas (`DiningTable`) distintas con `number` configurado (p. ej. Mesa 1 y Mesa 2) y
  su QR generable desde el panel de administración.
- Al menos un producto sin `image_url` y uno con `image_url` configurada, ambos activos y visibles
  en el catálogo del Menú QR.
- Un producto con más de un tamaño/variante, para Bug 4.

## Bug 1 — Invalidación de acceso tras cerrar sesión

1. Ejecutar los tests unitarios nuevos: `public-menu.component.spec.ts`,
   `diner-token.store.spec.ts` (comando de test de `pos-heladeria`, p. ej. `ng test` o el que use
   `@angular/build:unit-test`). Deben cubrir, como mínimo, FR-001 a FR-007.
2. Manual — flujo feliz: abrir la URL del QR de una mesa (`/menu/t/:token`), escribir un nombre,
   confirmar que se accede al menú. **Esperado**: acceso normal, sin cambios (Acceptance Scenario 1).
3. Manual — cerrar sesión: dentro del menú, pulsar "Salir"/"Cerrar sesión". **Esperado**: mensaje de
   despedida, sin acceso al menú/carrito.
4. Manual — recargar (F5) en la misma pestaña. **Esperado**: estado explícito de "acceso
   finalizado", **no** la pantalla de captura de nombre (FR-002).
5. Manual — pulsar "Atrás" del navegador. **Esperado**: no se restaura ninguna vista de
   menú/carrito autenticada (FR-003).
6. Manual — pulsar "Adelante" después de "Atrás". **Esperado**: mismo estado de "acceso finalizado",
   no se restaura el acceso (FR-004).
7. Manual — abrir la misma URL desde el historial del navegador (nueva pestaña **del mismo
   navegador**, pero reabriendo por historial/URL escrita a mano, no por escaneo): documentar el
   comportamiento observado frente al riesgo residual conocido de `spec.md` §Assumptions (puede
   variar según si el navegador reutiliza `sessionStorage` de una pestaña reabierta).
8. Manual — abrir la misma URL del QR en una pestaña **privada/incógnito nueva** (simula un
   contexto de navegación genuinamente distinto, equivalente en la práctica a un escaneo físico
   nuevo). **Esperado**: la pantalla de captura de nombre funciona con normalidad y se puede crear
   una sesión nueva (FR-006/SC-002) — este paso es el que prueba que el bloqueo NO deja el
   dispositivo inutilizable para siempre.

## Bug 2 — QR de mesas identificable y en dos tamaños

1. Ejecutar los tests unitarios nuevos: `table-qr.util.spec.ts`, `table-qr.component.spec.ts`.
   Deben cubrir FR-008 a FR-015.
2. Manual — panel de administración, modal de QR de la Mesa 1: descargar la variante "Mostrador" y
   la variante "Sticker". **Esperado**: dos archivos PNG distintos, ambos con "Mesa 1" visible.
3. Repetir con la Mesa 2. **Esperado**: ambos PNG (Mostrador y Sticker) muestran "Mesa 2", nunca
   "Mesa 1" ni un número de posición en una lista.
4. Escanear cada uno de los 4 PNG descargados (2 mesas × 2 variantes) con la cámara de un celular
   real. **Esperado**: los 4 abren la misma URL de destino que abría el QR de esa mesa antes de esta
   spec (mismo token, ver `contracts/README.md`) — verificar comparando la URL resuelta contra la
   generada por el flujo actual sin esta corrección.
5. Manual — generación masiva (`table-qr-sheet.component.ts`): generar el lote completo de mesas.
   **Esperado**: cada tarjeta impresa conserva el identificador correcto de su propia mesa
   (FR-015), sin regresión frente al comportamiento previo a esta spec.

## Bug 3 — Placeholder genérico en el catálogo

1. Ejecutar los tests unitarios nuevos/extendidos de `public-menu.component.spec.ts` que cubran
   FR-016 a FR-020.
2. Manual — abrir el Menú QR de una mesa y ubicar el producto sin `image_url`. **Esperado**: ícono
   genérico de "imagen no disponible", nunca el emoji `🍦`.
3. Manual — ubicar el producto con `image_url` configurada. **Esperado**: sigue mostrando su imagen
   real, sin cambios (FR-018).
4. Confirmar visualmente que el ícono nuevo sigue el mismo estilo (trazo único, mismo grosor) que
   el resto de íconos ya usados en la barra inferior/búsqueda del mismo menú (FR-019).

## Bug 4 — "Copiar insumos" solo con inventario activo

1. Ejecutar `product-form.component.spec.ts` extendido, cubriendo FR-021 a FR-026.
2. Manual — creación de producto con más de un tamaño: con el switch "maneja inventario" apagado,
   confirmar que el botón "Copiar insumos..." **no** aparece.
3. Manual — activar el switch sin recargar la página. **Esperado**: el botón aparece de inmediato
   (FR-022).
4. Manual — desactivarlo de nuevo. **Esperado**: el botón desaparece de inmediato (FR-023).
5. Manual — edición de un producto existente que ya tiene insumos guardados, con el switch apagado.
   **Esperado**: botón oculto, insumos siguen guardados (sin pérdida de datos, spec 027 sin
   cambios) — FR-024/FR-025.

## Resultado esperado del quickstart completo

Todos los pasos manuales anteriores producen el resultado "Esperado" indicado, y los comandos de
test unitario listados terminan en verde, antes de considerar esta spec lista para
`/speckit-tasks` → `/speckit-implement`.
