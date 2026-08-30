# Contrato: tipo de promoción `qty_price_presentation` y sus reglas

Extensiones a los endpoints **existentes** de `/promotions`
(`../pos-backend/app/api/v1/promotions/`). **Sin endpoints nuevos.** Sin cambio de forma en el PATCH
escalar, `duplicate`, `delete` ni `list`. Cubre el Incremento B (research.md D17).

Referencia de comportamiento heredado que NO cambia: spec 012 (motor de evaluación) y spec 013
(administración, máquina de estados, validación de forma, unicidad de nombre, `overlaps`
informativos).

---

## 1. Nuevo valor de `type`

`PromotionType` (`schemas.py`) y `PROMOTION_TYPES` (`models/promotion.py`) ganan
`"qty_price_presentation"`. `ck_promotions_type` ampliado (ver [data-model.md](../data-model.md)).

- **NO** entra en `AUTO_TYPES` — no participa del motor línea-por-línea (research.md D4), igual que
  `combo`.
- `Promotion.value` y `Promotion.min_qty` no se usan para este tipo (el precio y la cantidad viven
  por regla). El schema los deja en su default (`value=0`, `min_qty=1`) sin exigir nada.
- Máquina de estados idéntica (`PROMOTION_TRANSITIONS`): nace `draft` (o `active`/`paused`
  directamente, spec 013 FR-015), `finished` terminal.
- Cambio de forma (`type`, reglas) solo en `draft` (`update_shape`, spec 013 FR-004).

## 2. Payload de creación / cambio de forma

`PromotionCreate` y `PromotionShapeUpdate` ganan:

```text
presentation_rules?: [                       # requerido y no vacío cuando type == qty_price_presentation
  {
    presentation_id: UUID,                   # presentación existente y activa del tenant
    min_qty: int (>= 1),                     # cantidad mínima del paquete (1 es válido — CL-7)
    pack_price: Decimal (>= 0, 12,2)          # precio total del paquete
  }, ...
]
confirm_precio_no_uniforme?: bool = false    # override de FR-017 (ver §4)
confirm_sin_descuento?: bool = false         # override de FR-022 (ver §4)
```

`PromotionResponse` gana:

```text
presentation_rules: [
  {
    presentation_id: UUID,
    presentation_name: string,
    min_qty: int,
    pack_price: Decimal,
    applicable_variant_count: int            # variantes ACTIVAS que referencian esa presentación
                                             # (FR-005, "Productos Aplicables" / "Resumen")
  }, ...
]
```

## 3. Validación de forma

| Condición | Resultado | FR |
|---|---|---|
| `type == qty_price_presentation` sin `presentation_rules` o con lista vacía | 422 "Una promoción de precio por presentación necesita al menos una regla" | FR-001 |
| `presentation_rules` con `presentation_id` repetido | 422 "No puede haber dos reglas para la misma presentación" | FR-006 (1ª parte) |
| `presentation_rules` enviadas con `type != qty_price_presentation` | 422 "Las reglas por presentación solo aplican a promociones de ese tipo" | research.md D4 |
| `presentation_id` inexistente o inactiva | 422 "Presentación no encontrada o inactiva" | — |
| `min_qty < 1` o `pack_price < 0` | 422 (Pydantic) | FR-001, CL-7 |
| `targets` **y** `presentation_rules` a la vez | 422 "Una promoción de precio por presentación define su alcance en las reglas, no en targets" | research.md D3/D4 |
| Alguna regla apunta a una presentación ya cubierta por una regla de **otra** promoción `qty_price_presentation` con `status == active` — al **crear**, al **cambiar forma** o al **activar** (`PATCH /status {"status":"active"}`) | **409** `{ "error": "...", "conflicts": [{ "promotion_id": UUID, "promotion_name": string, "presentation_id": UUID }] }` | FR-006 (2ª parte), CL-4 |

El 409 de solape entre promociones **bloquea** (a diferencia del campo `overlaps` de spec 013
FR-025, que solo advierte). `overlaps` sigue devolviéndose informativo para el resto.

## 4. Verificaciones con confirmación explícita (solo al guardar la regla)

Corren **únicamente** en `POST /promotions` y `PATCH /promotions/{id}/shape` — nunca
retroactivamente (FR-018, CL-1/CL-1b): un cambio de precio posterior o una variante nueva no las
re-disparan.

### FR-017 — uniformidad de precio

Para cada regla, el servicio reúne las variantes **activas** que referencian su `presentation_id` y
sus precios vigentes.

```text
Si no todos los precios son iguales y confirm_precio_no_uniforme == false:
→ 422 {
    "error": "Los productos de la presentación no tienen el mismo precio",
    "presentation_id": UUID,
    "reference_unit_price": Decimal,          # el menor — el que se cobrará (FR-011)
    "variants": [{ "variant_id": UUID, "description": string, "price": Decimal }, ...]
  }

Si confirm_precio_no_uniforme == true: se guarda (nunca en silencio — sin el flag no pasa).
```

### FR-022 — la regla no representa un descuento real

```text
Si pack_price / min_qty >= reference_unit_price y confirm_sin_descuento == false:
→ 422 {
    "error": "El precio de paquete no representa un descuento",
    "presentation_id": UUID,
    "pack_unit_price": Decimal,               # pack_price / min_qty
    "reference_unit_price": Decimal
  }

Si confirm_sin_descuento == true: se guarda. En el cobro, el motor igual nunca deja una línea peor
que sin promoción (FR-023, ver contracts/cobro-y-preview.md).
```

El frontend muestra el detalle en un diálogo; al confirmar, reenvía el mismo payload con el flag
correspondiente en `true`.

## 5. `duplicate`

`POST /promotions/{id}/duplicate` (spec 013 FR-006) copia también `presentation_rules` (una fila
nueva por regla, sin compartir `id`), igual que ya copia `targets` y `combo_items`. La copia nace
`draft`; el solape de FR-006 se revalida al activarla, no al duplicar.

## 6. Frontend (`../pos-heladeria/src/app/modules/promotions/`)

- `promotion.interface.ts`: `PromotionType += 'qty_price_presentation'`; `PresentationRuleForm`
  (formulario), `PresentationRule` (respuesta, con `presentation_name` + `applicable_variant_count`)
  y `PresentationRuleIn` (payload); `PresentationConfirmFlags`; tipos de error
  `PresentationOverlapError` (409, FR-006) y `PresentationPriceCheckError` (422, FR-017/FR-022). El
  tipo **no** se agrega a `AUTO_TYPES` (paridad backend).
- `promotion.service.ts`: `create` / `updateShape` aceptan `presentation_rules` + los flags de
  confirmación; `submit()` enruta el 409 con `conflicts[]` a la señal `presentationConflicts` y el
  422 con `reference_unit_price` / `pack_unit_price` a la señal `presentationPriceCheck`.

### 6.1 Formulario dedicado de `qty_price_presentation` (crear + editar borrador)

`promotions-page.component.ts` — al **crear** (`@case ('presentation')` del asistente) **y** al
**editar un borrador** de este tipo (`@case ('edit')` con `isDraft()`) se muestra un formulario
propio, template `#presentationForm`, que reproduce la **estructura del mockup de `spec.md`
§Assumptions** con el estilo Tailwind actual de la app (índigo/gris, iconos SVG inline; **sin**
Material 3, Google Fonts ni Material Symbols — el `primary` del mockup ya coincide con
`indigo-600`). Layout `lg:grid-cols-3`.

**Columna izquierda:**

- **Tarjeta "Información General"** (icono + subtítulo "Detalles básicos para identificar la
  promoción"): Nombre (obligatorio); Descripción (opcional — el mockup no la trae, se conserva
  porque el backend la acepta); Fecha de inicio + Fecha de fin (opcionales, siempre visibles); Días
  de aplicación (pills, "Vacío = todos los días"); Hora de inicio + Hora de fin (opcionales, se
  admite cruzar la medianoche); Prioridad (pills Normal/Alta/Máxima tras "+ Más opciones" — el
  mockup la omite, el campo se conserva). → **FR-002 / FR-003 / FR-004**.
- **Tarjeta "Configuración de Reglas"** (acento primario: borde `border-2 border-indigo-200` +
  barra izquierda; subtítulo "Define el alcance y la mecánica del descuento"):
  - Aviso ámbar + enlace a `/dashboard/presentations` si no hay presentaciones activas.
  - Filas de regla (`grid md:grid-cols-3`): **Presentación** (`<select>` que **excluye las ya
    usadas** en otras filas — refuerzo de FR-006 1ª parte), **Cantidad** ("Al comprar" `N` "uds.",
    `min ≥ 1`, CL-7), **Precio total** (`$` + `app-money-input`). Botón de borrar al `hover` en la
    esquina superior derecha (solo si hay más de una regla).
  - Botón punteado "+ Añadir regla de presentación" (`disabled` cuando ya se usaron todas las
    presentaciones). → **FR-001 / FR-006 (1ª parte)**.

**Columna derecha:**

- **Panel "Productos Aplicables"** (icono `check-circle`): por cada regla con presentación elegida,
  "Aplica automáticamente a **N** producto(s) con la presentación **{nombre}**, sin importar su
  sabor o categoría (incluye los que se creen después)" — `N` = `applicable_variant_count`.
  → **FR-005 / FR-007 / FR-019**.
- **Panel "Resumen de la Regla"** (fondo `bg-indigo-50`, etiqueta uppercase, icono ✨): "Esta
  promoción aplicará las siguientes reglas:" + viñetas `• {min_qty}x {presentación}: {precio}` por
  cada regla completa. **Cumple FR-005** (resumen legible de todas las reglas antes de confirmar la
  creación o edición).

Flujo: crear → "Revisar y crear" → pantalla `review` (`draftSummary()` lista las reglas) →
"Guardar como borrador" / "Crear y activar". Editar borrador → "Guardar cambios". Una promoción
`qty_price_presentation` **ya activa/pausada/finalizada** se ve en modo **solo lectura** (lista
`min_qty × presentación por precio · N productos` desde `editingPromo().presentation_rules`), sin
cambios.

- **Diálogos de confirmación** para **FR-017** y **FR-022**: leen el detalle estructurado del 422
  (`variants[]` / `pack_unit_price`, `reference_unit_price`) y, al confirmar ("Guardar de todos
  modos"), reenvían el mismo payload con `confirm_precio_no_uniforme` / `confirm_sin_descuento` en
  `true`.
- **Diálogo del 409 de solape** (**FR-006** 2ª parte): nombra la(s) promoción(es) en conflicto.
- `promotion-pricing.util.ts`: `getPromoDisplay` reconoce el tipo para la insignia; queda fuera de
  la previsualización de precio (research.md D13, mismo trato que `qty_price`).

### 6.2 Retirada de `qty_price` de la creación

La tarjeta **"Paquete"** desaparece del selector "¿Qué quieres crear?" (`@case ('type')`): ya **no
se pueden crear** promociones `qty_price` desde la UI (`chooseKind` solo acepta `'discount'` y
`'presentation'`). Las promociones `qty_price` **existentes** se siguen listando, viendo y
**editando** con normalidad — el tipo sigue en `editableTypes` y la pantalla `@case ('pack')` +
`switchType('qty_price')` quedan intactas. **El backend no cambia**: `qty_price` sigue soportado en
el modelo, en `AUTO_TYPES`, en el motor y en la API.
