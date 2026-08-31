# Research: Notas del ítem visibles en "Mis pedidos" del Menú QR

## D1: Dónde y cómo renderizar `item.notes` en "Mis pedidos"

**Decision**: agregar un `@if (item.notes)` inmediatamente después del bloque `@if (optionLabels(item))`
ya existente (`public-menu.component.ts:245-247`), dentro del mismo `<li>` por ítem, usando la misma
clase visual (`class="block text-xs text-gray-400 pl-5"`) para que la nota se vea consistente con
las opciones y quede claramente asociada a su línea de ítem (spec.md FR-003).

```html
@if (optionLabels(item)) {
  <span class="block text-xs text-gray-400 pl-5">{{ optionLabels(item) }}</span>
}
@if (item.notes) {
  <span class="block text-xs text-gray-400 pl-5">📝 {{ item.notes }}</span>
}
```

**Rationale**:
- `item.notes` ya es un campo tipado directo (`string | null | undefined`), sin transformación
  necesaria — a diferencia de `optionLabels(item)`, que sí resuelve IDs de opción contra el
  catálogo (`this.lookup()`). No hace falta ningún método nuevo en la clase del componente.
- Repetir la nota bajo cada `<li>` de ítem (en vez de mostrarla una sola vez a nivel de pedido)
  resuelve FR-003: en un pedido con dos líneas idénticas de producto/opciones pero notas distintas,
  cada nota queda pegada a su propia línea, sin ambigüedad sobre cuál corresponde a cuál.
- Un prefijo visual corto (📝) distingue la nota de las opciones a simple vista sin necesitar un
  color o ícono nuevo del sistema de íconos compartido (`IconComponent`) — es texto libre del
  comensal, no una etiqueta estructurada como las opciones, así que basta un separador visual
  liviano dentro del `<span>` de texto, no un componente nuevo.
- Reutilizar la misma clase (`text-xs text-gray-400 pl-5`) evita cualquier decisión de diseño
  nueva: la nota se integra con la jerarquía visual que la tarjeta de pedido ya establece para
  información secundaria bajo el nombre del producto.

**Alternatives considered**:
- *Concatenar la nota dentro de `optionLabels(item)`* (mismo patrón que
  `pos-terminal.store.ts:789`, que junta opciones y nota en un solo string): rechazado porque
  `optionLabels(item)` ya es un método puro que resuelve nombres de opción por ID contra el
  catálogo (`this.lookup().optionLabel(...)`) — mezclarle la nota le cambiaría el propósito y
  obligaría a tocar su firma/comportamiento existente sin necesidad (Principio V, no refactors
  oportunistas). Mantenerlos como dos `@if` independientes es más simple y no toca código que ya
  funciona.
- *Mostrar la nota en un componente/badge separado con ícono de `IconComponent`*: rechazado por
  desproporcionado para el alcance — no hay ninguna otra pantalla que necesite reutilizar esa
  presentación (spec.md Assumptions), y el patrón de texto secundario bajo el producto ya es el
  establecido en esta misma tarjeta.
