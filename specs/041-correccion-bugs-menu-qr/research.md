# Research — Corrección de bugs y mejoras — Menú QR

Todas las decisiones técnicas de esta spec quedan resueltas aquí (no quedó ningún `NEEDS
CLARIFICATION` en el Technical Context del plan). No hay dependencias nuevas ni cambios de
`pos-backend` que investigar — la investigación se centra en el mecanismo exacto de cuatro
correcciones acotadas de `pos-heladeria`.

## Decisión 1 — Mecanismo de la marca de "acceso cerrado" (Bug 1)

**Decisión**: usar `sessionStorage` (no `localStorage`) para persistir que un acceso quedó
cerrado, con una clave que incluye el token QR de la mesa (p. ej. `pos.diner.exited_token`,
guardando el valor del token cerrado, siguiendo el mismo patrón try/catch de
`diner-token.store.ts` para navegación privada/storage bloqueado). `ngOnInit`
(`public-menu.component.ts:651-689`) consulta esa marca **antes** de decidir `view.set('name')`
en la línea 677: si el token actual de la URL coincide con el token marcado como cerrado, se
muestra un nuevo estado `view` explícito (p. ej. `'exited'`) en vez de `'name'`, sin ruta directa
desde ahí hacia `confirmName()`/`openSession()`.

**Rationale**: `sessionStorage` persiste dentro de la **misma pestaña** a través de recarga
(F5), "Atrás" y "Adelante" — cubre exactamente FR-002 a FR-005 sin ningún temporizador ni consulta
adicional al backend. A la vez, `sessionStorage` **no** se comparte con una pestaña nueva —ni
siquiera del mismo navegador y la misma URL—, que es, en la práctica, lo que produce un escaneo
físico nuevo desde la cámara del celular (la mayoría de navegadores móviles abren el enlace
escaneado en una pestaña/instancia nueva). Esto satisface FR-006/SC-002 (un intento de acceso
genuinamente nuevo sí debe funcionar) sin necesitar detectar criptográficamente que hubo una
cámara de por medio —algo que, como documenta `spec.md` §Assumptions, es imposible de garantizar
del lado del servidor dado que el token QR es permanente (regla protegida A-24)—. La decisión de
negocio (Clarifications, sesión 2026-08-27) exige que ni el tiempo transcurrido ni un cambio de
`table_session` levanten el bloqueo automáticamente; `sessionStorage` cumple eso por diseño, sin
código adicional para "no expirar": simplemente no tiene mecanismo de expiración temporal alguno.

**Alternatives considered**:
- **`localStorage`**: descartada — persiste indefinidamente entre **todas** las pestañas/ventanas
  del mismo navegador, así que una pestaña nueva abierta por un escaneo físico legítimo también
  quedaría bloqueada, violando FR-006/SC-002.
- **Bandera solo en memoria (propiedad de componente/servicio, sin storage)**: descartada — no
  sobrevive a una recarga real (F5), que es exactamente uno de los casos que FR-002 exige bloquear.
- **Token QR de un solo uso o rotado por escaneo**: descartada — cambiaría el contenido codificado
  en el QR físico impreso, violando FR-007/FR-012 y reabriendo la regla protegida A-24/`RN-CART-24`
  sin una nueva decisión de negocio explícita sobre esa regla (fuera de alcance, `spec.md` §Out of
  Scope).
- **Cookie no-`httpOnly`**: descartada — mismo alcance de escritura por JS que `sessionStorage`
  pero sin ninguna ventaja adicional para este caso, y el proyecto ya decidió (spec 007, A-21) no
  migrar el almacenamiento del token del comensal a cookies.

## Decisión 2 — Puntos de reingreso a auditar además de `ngOnInit` (Bug 1)

**Decisión**: además del chequeo en `ngOnInit`, revisar `expireSession()`
(`public-menu.component.ts:1164-1174`, que hoy hace `view.set('name')` ante cualquier `401`) para
que, si el token cuya sesión expiró corresponde a un acceso que el propio usuario cerró
explícitamente (no una expiración natural por inactividad, fuera de alcance por Edge Cases), no
reabra el camino hacia `'name'` sin pasar por la misma marca. `exit()`
(`public-menu.component.ts:1013-1032`) es el único punto que **escribe** la marca; se mantiene como
está en lo demás (sigue llamando a `POST /cart/leave` best-effort, sin cambios de FR-001).

**Rationale**: `expireSession()` es un segundo camino hacia `view.set('name')` que no pasa por
`ngOnInit` — si un cierre de sesión explícito dejara, por cualquier motivo, una petición en vuelo
que resuelve en `401` después de escribir la marca, ese handler no debe pisar la marca y reabrir
el formulario de nombre.

**Alternatives considered**: modificar únicamente `ngOnInit` — descartada como insuficiente porque
no cubre el camino de `expireSession()`, dejando un resquicio equivalente al bug original.

## Decisión 3 — Dimensiones de referencia de "Mostrador" y "Sticker" (Bug 2)

**Decisión**: componer el PNG final sobre un `<canvas>` 2D nativo, dibujando encima la imagen ya
generada por `QRCode.toDataURL` (`qrcode`, ya en uso) y el identificador de mesa como texto debajo,
dentro de una zona de seguridad. Valores de referencia:
- **Mostrador**: canvas de ~900×1100 px; módulo QR ~700 px de ancho (`QRCode.toDataURL(..., {
  width: 700, margin: 2 })`); banda de texto inferior con el identificador en fuente grande.
- **Sticker**: canvas de ~380×460 px; módulo QR ~300 px de ancho; banda de texto inferior con el
  identificador en fuente más pequeña pero legible.

Ambas variantes llaman a la misma función `menuUrlForToken`/token firmado que hoy usa
`qrDataUrl`/`buildTableQr` (`table-qr.util.ts:11-34`) — ningún cambio en el destino codificado.

**Rationale**: reutiliza el mismo parámetro `width`/`margin` que `qrDataUrl` ya expone hoy
(`table-qr.util.ts:16-18`), sin tocar la librería `qrcode` en sí. La proporción del módulo QR frente
al lienzo total (≈78-80%) deja espacio para el texto sin reducir el QR por debajo de un tamaño
cómodamente escaneable por una cámara de celular a la distancia típica de lectura de cada formato
(mostrador: se ve/lee a más distancia; sticker: se lee de cerca, pero en un área físicamente
pequeña). Estos valores son de referencia, ajustables durante la implementación siempre que se
respete FR-010/FR-011/FR-013 (Mostrador > Sticker, zona de seguridad, sin degradar la lectura).

**Alternatives considered**:
- **Generación del PNG en el backend** (p. ej. con una librería Python de imágenes): descartada —
  `pos-backend` no tiene hoy ninguna dependencia de generación de QR/imagen (`requirements.txt` sin
  `qrcode`/`Pillow` para este fin), agregarla exigiría justificar una dependencia nueva (Principio
  IX) para resolver algo que el frontend ya resuelve hoy sin backend.
- **Un único tamaño ajustable con control deslizante**: descartada — la spec pide explícitamente
  dos opciones nombradas y claramente diferenciadas ("Mostrador"/"Sticker"), no un tamaño libre.

## Decisión 4 — Ícono genérico "sin imagen" (Bug 3)

**Decisión**: agregar un caso nuevo `"image-off"` al `@switch (name)` ya existente en
`shared/icon/icon.component.ts`, con un SVG de trazo único (estilo Lucide, mismo `stroke-width` y
`viewBox` que los casos vecinos) que representa una imagen con una diagonal cruzándola (símbolo
estándar de "imagen no disponible"). `public-menu.component.ts:349` reemplaza el emoji `🍦` por
`<app-icon name="image-off" />`.

**Rationale**: sigue exactamente el patrón ya usado por el resto de íconos del componente (mismo
mecanismo de extensión, cero librerías nuevas, cero archivos de imagen nuevos) — el ícono
"image-off"/"imagen tachada" es un símbolo ampliamente reconocido para "sin imagen" en el estilo
Lucide/Feather que el resto del componente ya imita.

**Alternatives considered**: un archivo SVG/PNG estático en `assets/` — descartada porque el
proyecto no usa ese patrón para íconos (todos viven como casos del `@switch` en
`icon.component.ts`), y agregar un archivo rompería esa convención sin necesidad.

## Decisión 5 — Condición del botón "Copiar insumos..." (Bug 4)

**Decisión**: extender la condición existente en el template de
`product-form.component.ts:384-389` de `draft().hasSizes && draft().variants.length > 1` a
`draft().hasSizes && draft().variants.length > 1 && draft().tracks_inventory` — sin tocar
`copyConfigToOthers()` (`:858-881`) ni ningún otro método, porque la operación ya es puramente en
memoria hasta `save()` (`:1067`, FR-026) y no existe endpoint propio que además necesite validar el
switch.

**Rationale**: `draft().tracks_inventory` ya es la señal que gobierna la sección de insumos completa
(`:244`, `:378-382`) — reutilizar exactamente esa misma señal en la condición del botón es la
extensión mínima y coherente con el patrón de signals ya usado en todo el formulario.

**Alternatives considered**: ninguna — no hay ambigüedad técnica en este bug (confirmado en la
investigación previa a la spec; sin research adicional necesario).
