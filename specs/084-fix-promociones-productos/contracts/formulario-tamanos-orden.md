# Contrato: Formulario de producto — tabla de tamaños sin "Nombre" y orden de la tarjeta (enmienda 2026-09-20)

Cubre spec.md FR-029 (parte de UI), FR-037 a FR-039 (User Story 7 parte frontend y User Story 8).
Solo frontend: `pos-heladeria/src/app/modules/products/pages/product-form.component.ts`, sin cambio
de API propio (el de datos está en [variante-sin-nombre.md](./variante-sin-nombre.md)).

## Orden de la tarjeta "Tamaños del producto" (US8)

Hoy (línea ~160-286 del template), con tamaños encendidos:

```
Encabezado + interruptor de tamaños
Maneja inventario (interruptor)  ← aquí queda entre el encabezado y la tabla
[aviso "no podrá venderse…"]
Tabla de tamaños  (+ Agregar tamaño)
Presentaciones desactivadas
Detalle del tamaño activo (Insumos fijos, Sabores a elegir)
```

Después:

```
Encabezado + interruptor de tamaños
Tabla de tamaños  (+ Agregar tamaño)          ← @if (draft().hasSizes)
Presentaciones desactivadas                    ← sigue pegada a la tabla
Maneja inventario (interruptor)
[aviso "no podrá venderse…"]
Detalle del tamaño activo (Insumos fijos, Sabores a elegir)
```

- Con tamaños apagados no hay tabla ni lista de desactivadas: el orden queda `Encabezado →
  Maneja inventario → aviso → detalle (precio, insumos, sabores)`, igual que hoy.
- Es un **movimiento de bloques** del template; no cambia estado, señales, ni qué habilita cada
  interruptor (`sectionsEnabled()`, `showsInventoryWarning()`, `inventarioIncluido()` intactos).
- El separador visual (`mt-4 pt-4 border-t border-gray-100`) que hoy lleva "Maneja inventario"
  respecto del encabezado se conserva pero ahora separa "Maneja inventario" de lo que quede encima
  (tabla o lista de desactivadas); no debe quedar un `border-t` doble ni un margen huérfano.
- El texto del recuadro gris del detalle ("Activa «Maneja inventario» arriba para configurar los
  insumos fijos…") **sigue siendo correcto**: el interruptor sigue arriba del detalle.

## Tabla de tamaños (US7)

- Columnas: `⠿` · `#` · **Presentación** · **Precio** · acción. Se elimina la columna "Nombre"
  (encabezado y celda `<input>`). El grid pasa de `[28px_28px_1fr_150px_140px_88px]` a
  `[28px_28px_1fr_140px_88px]`; el `<select>` de presentación ocupa el `1fr` (antes 150px).
- `<select>`: **sin** opción "Sin presentación". Con `presentationId === null` (fila nueva) muestra
  una opción deshabilitada "Elige una presentación". Las opciones siguen siendo las activas más la
  que la fila ya tenga aunque esté inactiva (`presentationOptionsFor(v)`, hoy solo filtra por
  `active`), y **como cambio nuevo** excluyen las ya elegidas en otras filas del mismo producto:
  sin la columna "Nombre" el administrador ya no tiene otra pista visual de un duplicado, y el 409
  de FR-006 queda como respaldo. Si eso deja la lista vacía se muestra "No hay más presentaciones
  disponibles — créalas en Presentaciones".
- Validación: se agrega a `canSave()` (hoy ~l.598, ya condiciona el botón Guardar) la condición
  "con tamaños encendidos, toda fila tiene `presentationId`"; la fila incompleta muestra bajo su
  `<select>` el texto "Elige una presentación" (mismo patrón inline que `groupError`). Con tamaños
  apagados no aplica: la variante única va con `presentation_id: null` ⇒ el backend usa
  "Presentación única" (FR-034).
- `setVariantPresentation()` deja de tocar `name` (solo asigna `presentationId` y
  `presentationName`). `setVariantField(..., 'name', ...)` se elimina.
- **Encender tamaños** (`toggleHasSizes`, antes sembraba `name: 'Grande'`/`'Mediana'`/`'Pequeña'`):
  se siguen creando 3 filas que heredan precio, receta y grupos de la variante actual, y a cada
  una se le **preselecciona** la presentación activa del catálogo de ese nombre (Grande, Mediana,
  Pequeña/Pequeño…) si existe; si no existe, la fila queda sin elegir. Si la variante actual ya
  tenía una presentación real distinta de "Presentación única", esa fila la conserva. **Apagar**:
  se conserva la primera fila y pasa a "Presentación única" (`presentationId: null`, el backend la
  resuelve). Se eliminan los literales `'Grande'`/`'Único'` como nombres.
- `addVariant()` (hoy ~l.873): fila nueva con `presentationId: null`, sin `name`.
- "Presentaciones desactivadas": la lista muestra `dv.presentationName` (antes `dv.name`).
  Restaurar (`restoreVariant`) sigue igual.
- El panel de detalle usa `activeVariant()`; el único texto que hoy interpola `av.name` (l.~463,
  "Copiar insumos y sabores de «…» a los otros tamaños") pasa a `av.presentationName`, o a
  "este tamaño" si la fila aún no tiene presentación.
