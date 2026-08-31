# Quickstart: Notas del ítem visibles en "Mis pedidos" del Menú QR

Valida la Historia de Usuario 1 (spec.md) — el comensal ve, en su propio pedido, la nota que
escribió al agregar un ítem al carrito.

## Prerrequisitos

- `../pos-heladeria` con dependencias instaladas (`npm install`).
- No requiere `../pos-backend` corriendo para la validación automatizada (test unitario con
  `FakeDinerService`); sí lo requiere la validación manual end-to-end.

## Validación automatizada

```bash
cd ../pos-heladeria
npx ng test --include='**/public-menu.component.spec.ts'
```

Caso nuevo esperado en `public-menu.component.spec.ts` (sección "Mis pedidos"):

1. Un pedido con dos ítems idénticos en producto/opciones, uno con `notes: 'sin banana'` y otro sin
   nota (`notes: null`), renderizado en `section() === 'pedidos'`.
2. Verifica que el texto `'sin banana'` aparece en el DOM.
3. Verifica que solo aparece una vez (asociado a su línea, no duplicado ni mostrado para el ítem
   sin nota).

## Validación manual end-to-end

1. Levantar `pos-backend` y `pos-heladeria` en local (o usar el ambiente de staging existente).
2. Escanear/abrir el Menú QR de una mesa de prueba, iniciar sesión de comensal.
3. Agregar un producto al carrito y escribir una nota (p. ej. "sin banana") antes de enviarlo.
4. Enviar el pedido.
5. Ir a la pestaña "Mis pedidos".

**Resultado esperado**: la nota "sin banana" aparece bajo la línea de ese producto, con el mismo
estilo visual que ya usan las opciones seleccionadas. Un ítem enviado sin nota no muestra ningún
elemento adicional bajo su línea.

**Riesgo a verificar**: repetir el paso 3-5 con dos unidades del mismo producto en el mismo pedido,
una con nota y otra sin nota (o con una nota distinta), y confirmar que cada línea muestra su
propia nota sin mezclarse con la otra (spec.md, Acceptance Scenario 4).
