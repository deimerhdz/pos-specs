# Contrato: Menú de acciones sin desplazar la tabla (bug 5)

Cubre spec.md FR-026 a FR-028. **Sin cambio de backend.** Alcance confirmado a un solo archivo
(research.md D8): `promotions-page.component.ts` — `products-page.component.ts` no tiene este
patrón (usa botones inline, no dropdown).

## Comportamiento actual (a reemplazar)

`promotions-page.component.ts:284-354`: menú ⋮ como `<div class="absolute right-0 mt-1 w-44 ...">`
dentro de `<div class="relative inline-block ...">`, dentro de `<td class="... relative">`, dentro
de `<div class="overflow-x-auto">` (línea ~249). `toggleActionsMenu`/`closeActionsMenu`
(línea ~1429-1436) solo manejan un signal `openActionsId`, sin manejo de scroll/resize. El
`overflow-x: auto` del contenedor fuerza `overflow-y: auto` (regla del spec CSS de `overflow`),
recortando el menú cuando la fila está cerca del borde.

## Comportamiento nuevo

Reemplazar el `<div>` absoluto por Angular CDK Overlay (`@angular/cdk/overlay`, ya instalado, sin
uso previo en el repo — research.md D5):

- El botón ⋮ de cada fila se convierte en el *origin* de un `CdkOverlayOrigin` (o se usa
  `cdkConnectedOverlay` directamente sobre el botón).
- El panel de opciones (Configurar/Duplicar/Pausar-Activar/Eliminar) se renderiza vía
  `Overlay.create()`/`CdkConnectedOverlay` en el `cdk-overlay-container` global — fuera del árbol
  DOM de la tabla, sin quedar sujeto a su `overflow`.
- Posición: conectada al botón origen (`connectedPosition`), con flip automático si no cabe hacia
  abajo (comportamiento estándar de CDK, sin código manual).
- Cierre: al hacer clic fuera (`hasBackdrop` + `backdropClick()`), al seleccionar una opción, o al
  hacer scroll de la tabla mientras está abierto — CDK Overlay expone `scrollStrategy` (p. ej.
  `close()` o `reposition()`); dado que spec.md FR-028 permite cerrar **o** reposicionar, se usa la
  estrategia `close()` por simplicidad (evita tener que recalcular la posición del botón origen en
  cada evento de scroll dentro de una tabla con su propio contenedor de scroll).
- Resize de ventana: mismo criterio — se cierra el overlay abierto (comportamiento por defecto de
  CDK ante cambios de layout que invalidan la posición calculada, spec.md Edge Cases).

## Casos de aceptación cubiertos

- Fila cerca del borde inferior/superior del contenedor con scroll → menú visible completo, sin
  recorte, sin generar una barra de scroll adicional dentro de la tabla.
- Clic fuera o selección de una opción → menú se cierra, posición de scroll de la tabla sin cambio.
- Scroll de la tabla con el menú abierto → el menú se cierra (estrategia `close()`), nunca queda
  flotando sobre una fila distinta a la que lo abrió.
- Resize de ventana con el menú abierto → se cierra.
