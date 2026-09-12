# Quickstart: Pestaña dedicada de Promociones en el menú QR

Validación ejecutable de spec.md por historia de usuario. No requiere datos nuevos más allá de
una promoción vigente ya creada por la administración existente (specs 013/063/071) — esta
funcionalidad no cambia cómo se crean.

## Prerrequisitos

1. `pos-backend` corriendo localmente: `uvicorn app.main:app --reload` (puerto por defecto).
2. `pos-heladeria` corriendo localmente: `npm start` (`ng serve`).
3. Un tenant de prueba con al menos:
   - Un producto **A** con una única variante, cubierta por una regla **vigente** de tipo
     "precio de paquete" con `min_qty = 2` (ej. "2 x $17.000") — créala en
     `/dashboard/promotions` (administración de promociones, sin cambios de esta spec).
   - Un producto **B** en la misma o distinta categoría, **sin** ninguna promoción vigente.
   - (Opcional, para US1 acceptance scenario 2 y el contrato §1) Un producto **C** cubierto por
     una regla de tipo "porcentaje" con `min_qty = 3`, para confirmar que FR-004 también aplica a
     ese tipo (Clarifications, sesión 2026-09-12, pregunta 2).
4. Abre el menú QR de una mesa de ese tenant (`/m/{token}` o el flujo de escaneo habitual).

Corre la suite de characterization tests de backend **antes** de tocar código, como línea base
(debe estar en verde; esta spec no la modifica):

```bash
cd ../pos-backend
python -m unittest discover -s app/characterization_tests -p 'test_*.py' -v
```

## Historia 1 — Encontrar todos los productos en promoción (P1)

Ver contrato [pestana-promociones-comportamiento.md §1](./contracts/pestana-promociones-comportamiento.md#1-elegibilidad-de-producto-fr-002-fr-008--cubre-us1).

1. Con el menú QR abierto, confirma que aparece una pestaña **"Promociones"** junto a las
   categorías existentes (FR-001).
2. Selecciónala. Verifica que solo aparecen el producto **A** y el producto **C** (si lo creaste),
   sin importar su categoría original, y que el producto **B** no aparece (FR-002).
3. Abre la tarjeta de **A**: la condición de la promoción (tipo + cantidad mínima) se ve igual
   que en su categoría original (FR-003).
4. Pausa o vence la promoción de **A** y **C** (o crea un tenant de prueba sin ninguna vigente) y
   vuelve a seleccionar "Promociones": debe verse el aviso "No hay promociones activas en este
   momento" en vez de una lista vacía sin explicación (FR-008), y la pestaña sigue visible.
5. Vuelve a una categoría normal: sigue mostrando todos sus productos, tengan o no promoción
   (FR-009) — sin cambio frente al comportamiento anterior a esta spec.

**Éxito**: los 5 pasos verifican SC-001.

## Historia 2 — Agregar respetando la cantidad mínima (P1)

Ver contrato [§2](./contracts/pestana-promociones-comportamiento.md#2-paso-de-cantidad-al-agregar-por-primera-vez-fr-004-fr-005-fr-012--cubre-us2)
y [§3](./contracts/pestana-promociones-comportamiento.md#3-paso-de-cantidad-en-el-carrito-ya-creado-fr-004-fr-006--cubre-us2-us3).

1. Desde la pestaña "Promociones", abre **A** (regla "2 x $17.000", `min_qty = 2`) y confírmalo
   sin tocar el selector de cantidad. Verifica en el carrito: cantidad = **2**, precio de paquete
   ya aplicado (FR-004, FR-007) — no cantidad 1.
2. Abre el carrito y presiona "+" sobre esa línea: la cantidad pasa a **4**, nunca a 3 (FR-004).
3. Presiona "−" dos veces seguidas desde 4: primero baja a **2** (un paso completo, no a 3),
   luego retira la línea del carrito por completo (no queda en 1 unidad) (FR-006).
4. Repite el paso 1 con el producto **C** (regla "porcentaje", `min_qty = 3`): la cantidad debe
   quedar en **3**, confirmando que el paso no depende del tipo de regla (FR-004, Clarifications
   pregunta 2).
5. Abre cualquier producto con una regla vigente de `min_qty = 1` (o `discounted_price` ya
   poblado en el menú) desde "Promociones": se agrega de a una unidad, igual que hoy (FR-005).
6. Caso de ajuste al múltiplo (FR-012, contrato §2): agrega **A** desde su categoría original
   (no desde "Promociones") — queda en cantidad 1, sin descuento pleno (comportamiento actual).
   Ahora ábrelo de nuevo desde "Promociones": la cantidad que propone el modal al abrir es **1**
   (no 2), porque solo hace falta 1 unidad más para completar el paquete de 2; confirma y
   verifica que el total en el carrito queda en **2**, no en 3.
7. Caso de cambio de variante dentro del modal (FR-011, hallazgo C1 de `/speckit-analyze`): en un
   producto con al menos dos variantes donde solo una está cubierta por una regla vigente
   (`min_qty = 2`), ábrelo desde "Promociones" con la variante **sin** promoción preseleccionada
   (cantidad debe mostrar 1). Cambia la selección a la variante **con** promoción sin cerrar el
   modal: la cantidad debe reajustarse de inmediato a **2**, sin necesidad de tocar "+"/"−".

**Éxito**: los 7 pasos verifican SC-002 y SC-003.

## Historia 3 — Las categorías normales no cambian (P2)

Ver contrato [§3, último párrafo](./contracts/pestana-promociones-comportamiento.md#3-paso-de-cantidad-en-el-carrito-ya-creado-fr-004-fr-006--cubre-us2-us3).

1. Sin pasar por "Promociones", agrega **A** desde su categoría original: queda en cantidad 1
   (FR-010) — sin el salto a 2 de la Historia 2, paso 1.
2. Con esa línea en el carrito, presiona "+": sube a 2 de a 1 en 1 (paso libre, no forzado a
   saltar de 2 en 2) — confirma que el descuento de paquete solo se refleja cuando la cantidad
   acumulada alcance el mínimo, exactamente como ya funcionaba antes de esta spec.
3. Presiona "−" desde 2: baja a 1, no se retira la línea todavía — comportamiento libre normal.

**Éxito**: los 3 pasos verifican SC-004 (sin regresión).

## Verificación cruzada final

1. Vuelve a correr la suite de characterization tests de backend — debe seguir en verde, sin
   ninguna diferencia frente a la línea base (research.md D6, Constitution Check §III):

   ```bash
   cd ../pos-backend
   python -m unittest discover -s app/characterization_tests -p 'test_*.py' -v
   ```

2. Corre las pruebas de frontend afectadas:

   ```bash
   cd ../pos-heladeria
   npm test -- --run public-menu.component.spec.ts product-select.component.spec.ts \
     dining-cart.service.spec.ts cart.component.spec.ts
   ```

3. Confirma manualmente que ninguna otra superficie que consuma `GET /menu` o `menu-lookup.ts`
   (Terminal de mesas, cocina, administración) cambió de comportamiento — no deberían, porque
   ningún archivo de `pos-backend` ni de `menu-lookup.ts` fue tocado (Constitution Check, gate II).
