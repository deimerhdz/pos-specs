# Quickstart: Validación de Mejora de Visibilidad del Buscador y Filtros en Inventario

Guía para verificar manualmente, de punta a punta, que la mejora visual cumple los
criterios de aceptación de [spec.md](./spec.md). No requiere datos de prueba especiales:
basta con un tenant que tenga al menos un par de insumos cargados (para que la tabla y la
tarjeta "Bajo mínimo" tengan contenido real que comparar visualmente).

Referencias de diseño: mockup de referencia visual exacta en
[design/inventory-search-filters-redesign.html](./design/inventory-search-filters-redesign.html)
(subárbol `data-purpose="table-card-header"`) y decisiones de tratamiento visual en
[research.md](./research.md). No hay contratos de API ni modelo de datos involucrados — ver
[data-model.md](./data-model.md).

> **Nota**: si al validar encuentras la zona de búsqueda y filtros como una tarjeta
> **separada** de la tabla con acento `indigo` genérico (en vez de fusionada en la misma
> tarjeta con acento `#4a3aff`), el componente todavía tiene la iteración anterior de este
> spec — ver "Estado actual del código" en research.md — y falta re-implementar contra el
> diseño vigente antes de poder validar los pasos siguientes.

## Prerrequisitos

- Rama `feat/085-improve-inventory-filters-visibility` en checkout dentro de `pos-heladeria`.
- `pos-backend` corriendo (o el ambiente que use normalmente `pos-heladeria` para consumir
  la API) con al menos un tenant que tenga insumos cargados, algunos por debajo del mínimo
  (para poder ver la tarjeta "Bajo mínimo" activa).
- Dependencias instaladas: `npm install` en `pos-heladeria` si no se hizo antes.

## 1. Arrancar la app y ejecutar la suite existente

```bash
cd pos-heladeria
npm test        # ng test — debe seguir en verde, sin modificaciones a
                 # inventory-page.component.spec.ts (Principio X / FR-005)
npm start        # o `ng serve` — levanta la app localmente
```

## 2. Historia 1 — Ubicar la zona de búsqueda y filtros de un vistazo

1. Iniciar sesión y navegar a **Inventario → pestaña "Insumos"**.
2. Con la página ya cargada, mirar la pantalla completa de corrido (sin buscar
   deliberadamente el buscador): confirmar que la zona de búsqueda y filtros se distingue a
   simple vista del encabezado, de la tarjeta "Bajo mínimo" y de la tabla de resultados
   (Acceptance Scenario 1).
3. Comparar la zona de filtros con sus vecinos inmediatos (tarjeta "Bajo mínimo" arriba,
   tabla abajo): confirmar que comunica claramente "esto es interactivo", no un bloque de
   contenido estático más (Acceptance Scenario 2).
4. **Idealmente**, repetir el paso 1 con 2-3 personas que no conozcan la pantalla,
   cronometrando cuánto tardan en señalar el campo de búsqueda → **SC-001** (≥90% en menos
   de 3 segundos). Si no hay personas disponibles para la prueba formal, al menos confirmar
   que el propio ubicarlo no requiere esfuerzo.

## 3. Historia 2 — Distinguir el buscador de los filtros entre sí

1. Ya ubicada la zona, observar específicamente el campo de texto: confirmar que se percibe
   como "un lugar para escribir" (borde, ícono de lupa u otra señal — no solo un rectángulo
   ambiguo) (Acceptance Scenario 1).
2. Observar los selectores de "Tipo de insumo" y "Estado": confirmar que cada uno se percibe
   como un control de filtro independiente, separado del buscador y entre sí (Acceptance
   Scenario 2).
3. **Idealmente**, repetir con las mismas personas del paso 2.3 pidiéndoles que señalen cada
   control por separado → **SC-002** (≥90% en menos de 5 segundos).

## 4. Edge cases

- **Ventana angosta**: reducir el ancho del navegador (o usar el modo responsive de las
  devtools, ~375px) hasta que el buscador y los filtros se acomoden en varias líneas.
  Confirmar que la zona sigue siendo igual de reconocible en ese acomodo (FR-006).
- **"Bajo mínimo" activo simultáneamente**: hacer clic en la tarjeta "Bajo mínimo" para
  activarla como filtro y dejarla en ese estado. Confirmar que la zona de búsqueda y
  filtros se sigue distinguiendo igual de bien, sin competir visualmente con la tarjeta ni
  perderse junto a ella.
- **Resto de la pantalla sin degradar**: confirmar que la tarjeta "Bajo mínimo" y la tabla
  de resultados conservan exactamente su apariencia y jerarquía visual actuales (FR-008) —
  comparar contra una captura de la rama `main` si hay dudas.
- **Sin depender solo del color**: con la emulación de daltonismo del panel "Rendering" de
  Chrome DevTools (o una captura pasada a escala de grises), confirmar que la zona de
  búsqueda y filtros se sigue distinguiendo del resto de la pantalla apoyándose en borde,
  espaciado e ícono de lupa — sin depender del tono morado `#4a3aff` (FR-004; Edge case de
  baja visión de spec.md).
- **Elementos excluidos por FR-009**: confirmar que la zona reforzada **no** incluye el
  badge "⌘K" dentro del campo de búsqueda ni un contador tipo "N Insumos" junto a los
  filtros — ambos aparecen en el mockup de referencia pero fueron excluidos explícitamente
  por la Clarification Q2 de spec.md.

## 5. Contraste (SC-005, FR-004)

1. Con las devtools del navegador (por ejemplo, el inspector de contraste de Chrome/Edge o
   la extensión axe), medir el contraste texto/fondo de: el texto ingresado en el buscador,
   el placeholder "Buscar por nombre...", el texto de las opciones de los selects, y
   cualquier texto nuevo (labels) que se agregue a la zona.
2. Confirmar que todas las combinaciones cumplen como mínimo **WCAG 2.1 AA**
   (4.5:1 para texto normal, 3:1 para texto grande/elementos gráficos).

## 6. Cero regresión funcional (SC-004, FR-005, FR-007)

Con la zona ya reforzada visualmente, confirmar que el comportamiento es idéntico al de
antes del cambio:

1. Escribir un término en el buscador → la tabla filtra por nombre igual que antes (mismo
   debounce, mismos resultados).
2. Cambiar el selector de "Tipo de insumo" → la tabla filtra igual que antes.
3. Cambiar el selector de "Estado" → la tabla filtra igual que antes.
4. Combinar búsqueda + ambos filtros + "Bajo mínimo" activo → el resultado combinado es
   idéntico al que se obtenía en `main` antes de este cambio.
5. Confirmar que la paginación de la tabla (`app-pagination-bar`) sigue funcionando sin
   cambios.

## 7. Confirmación del usuario que reportó el problema (SC-003)

Mostrar el cambio ya implementado (local o en el ambiente que corresponda) al usuario que
reportó originalmente que "no distinguía dónde estaba esa sección", y registrar su
confirmación explícita de que ahora la ubica con claridad de un vistazo. Este es el
criterio de éxito principal del spec, según sus propias Assumptions (validación
predominantemente cualitativa).

## Resultado esperado

Todos los puntos anteriores se cumplen sin que la suite automatizada (`npm test`) se vea
afectada, sin cambios de comportamiento de búsqueda/filtrado, y sin degradar la tarjeta
"Bajo mínimo" ni la tabla de resultados.
