---

description: "Task list for spec 060 — copiar número de cuenta/celular al portapapeles y descargar la imagen del QR en el paso 'Datos de transferencia' del checkout del comensal"
---

# Tasks: Copiar número de cuenta y descargar QR en pagos por transferencia

**Input**: Design documents from `/specs/060-copiar-cuenta-descargar-qr/`
**Prerequisites**: plan.md, spec.md, research.md, quickstart.md (sin data-model.md ni contracts/ —
no aplican a esta spec, ver plan.md)

**Tests**: incluidos — mismo criterio que specs 054-058 (Principio III/X de la constitución). Este
componente y `icon.component.ts` no tienen ningún test hoy (research.md D5), así que todos los
casos de esta spec son nuevos, no ajustes sobre tests existentes.

**Organización**: por historia de usuario de spec.md (US1/US2, en el orden en que aparecen). Todo
el código vive en el repositorio hermano `../pos-heladeria` (no en `pos-specs`); sin cambios en
`../pos-backend`.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: puede ejecutarse en paralelo (archivo distinto, sin dependencias pendientes). Las dos
  historias comparten el mismo archivo de test nuevo
  (`transfer-details-step.component.spec.ts`, research.md D5) y el mismo componente de producción
  (`transfer-details-step.component.ts`) — esas tareas no llevan `[P]` entre sí para evitar
  conflictos de merge sobre las mismas líneas. Cada historia sí toca `icon.component.ts` de forma
  aislada (un `@case` nuevo cada una, sin solapar líneas con la otra ni con
  `transfer-details-step.component.ts`), así que esas dos tareas concretas llevan `[P]`.
- **[Story]**: US1 o US2 — solo en fases de historia de usuario

---

## Phase 1: Setup

- [X] T001 Confirmar el baseline exacto: ejecutar `ng test --watch=false` dentro de
      `pos-heladeria` y verificar 573/577 tests en verde con exactamente estos 4 fallos
      preexistentes y ajenos a esta spec, sin ningún otro fallo inesperado:
      `auth.service.spec.ts` ("clears mustChangePassword..."),
      `sidebar.component.spec.ts` ("shows super admin navigation..."),
      `pos-checkout-panel.component.spec.ts` ("T032: ofrece 'Imprimir Pre-cuenta'...", ya
      documentado en spec 058 tasks.md T001) — y confirmar que ni
      `transfer-details-step.component.ts` ni `icon.component.ts` tienen ningún archivo de test
      hoy (research.md D5)

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: crear el archivo de test nuevo con su harness base — ambas historias agregan casos a
este mismo archivo, así que el `TestBed` y los mocks compartidos deben existir antes de escribir
el primer caso de cualquiera de las dos.

**⚠️ CRITICAL**: ninguna tarea de test de US1 o US2 puede empezar hasta que esta fase esté completa

- [X] T002 Crear
      `pos-heladeria/src/app/modules/tables/pages/checkout/transfer-details-step.component.spec.ts`
      con el harness base, sin ningún caso de prueba todavía: configuración `TestBed` standalone
      para `TransferDetailsStepComponent`, con mocks/spies de `DinerService.getPaymentMethods`,
      `DinerTokenStore`, `DiningCartService`, `CheckoutProgressStore` (`read()` devolviendo
      `{ payment_method_id: 'm1' }`, `paymentMethods()`/`paymentMethods.set()`), `Router`
      (`navigate` espiado), `ActivatedRoute` (`snapshot.paramMap.get('token')` devolviendo un token
      de prueba) y `ToastService` (spies de `success`/`error`, ya que la Fase 3/4 lo van a inyectar
      en el componente); agregar un helper `buildMethod(fields: PaymentMethodField[], paymentInfo:
      Record<string, string>): DinerPaymentMethod` para construir métodos de prueba con campos de
      texto y/o imagen configurables (research.md D5, D7)

**Checkpoint**: harness listo — los tests de US1 y US2 pueden empezar en paralelo si se desea.

---

## Phase 3: User Story 1 - Copiar el número de cuenta/celular al portapapeles (Priority: P1) 🎯 MVP

**Goal**: cada campo de texto que el método de pago ya muestra (`textFields()`) gana un ícono de
copiar que escribe su valor exacto al portapapeles, con una notificación no bloqueante de 5000 ms
en éxito o error (FR-001, FR-002, FR-003, FR-008, FR-010 y FR-011 para campos de texto).

**Independent Test**: abrir "Datos de transferencia" de un método con un campo de texto
configurado, tocar el ícono de copiar, pegar en otro campo y confirmar que el valor coincide
exactamente; verificar la notificación de éxito de 5s (quickstart.md Escenario 1).

### Tests for User Story 1

- [X] T003 [US1] En `transfer-details-step.component.spec.ts`: caso que, con un método construido
      vía `buildMethod` con 2 campos de texto con valor, verifica que se renderiza un
      botón/ícono de copiar identificable (p. ej. `aria-label`/`title` de "Copiar") junto a **cada
      uno** de los dos campos (FR-002, FR-010)
- [X] T004 [US1] En el mismo archivo: caso que simula clic en el botón de copiar de un campo,
      mockeando `navigator.clipboard.writeText` para que resuelva con éxito, y verifica que (a) se
      llamó con el valor exacto mostrado en pantalla (sin espacios ni texto adicional) y (b) se
      llamó `toast.success(...)` con `5000` como segundo argumento (FR-001, FR-003, FR-007 para
      éxito)
- [X] T005 [US1] En el mismo archivo: caso que mockea `navigator.clipboard.writeText` para que
      rechace (simula permiso denegado), simula el mismo clic, y verifica que se llamó
      `toast.error(...)` con `5000` como segundo argumento y que `toast.success` **no** se llamó
      (FR-008, FR-007 para error)
- [X] T006 [US1] En el mismo archivo: caso que, con un método construido sin ningún campo de texto
      con valor (solo un campo de imagen), verifica que no aparece ningún botón/ícono de copiar en
      la pantalla (FR-011, primera mitad)

### Implementation for User Story 1

- [X] T007 [US1] En
      `pos-heladeria/src/app/modules/tables/pages/checkout/transfer-details-step.component.ts`:
      inyectar `ToastService` (`private readonly toast = inject(ToastService)`) y agregar
      `async copyField(value: string): Promise<void>`, que llama
      `await navigator.clipboard.writeText(value)` dentro de un `try/catch`, notificando
      `this.toast.success('Copiado al portapapeles', 5000)` en éxito o
      `this.toast.error('No se pudo copiar', 5000)` en el `catch` (research.md D1, D6)
- [X] T008 [P] [US1] En `pos-heladeria/src/app/shared/icon/icon.component.ts`: agregar el caso
      `@case ('copy')` con el trazo SVG estándar de "copiar" (dos rectángulos superpuestos), mismo
      estilo *single-stroke* (`stroke="currentColor"`) que el resto del `@switch` (research.md D4)
- [X] T009 [US1] En `transfer-details-step.component.ts`: dentro del `@for (f of textFields();
      track f.key)` (líneas ~57-61), agregar un botón junto al valor
      (`(click)="copyField(method()!.payment_info?.[f.key] ?? '')"`) con
      `<app-icon name="copy">` y un `aria-label`/`title` de "Copiar" (research.md D7). Depende de
      T007, T008. Hace pasar T003, T004, T005, T006.
- [ ] T010 [US1] Ejecutar manualmente quickstart.md Escenario 1 (UI)

**Checkpoint**: copiar el número de cuenta/celular funciona de punta a punta, verificable de forma
independiente de US2.

---

## Phase 4: User Story 2 - Descargar la imagen del código QR (Priority: P2)

**Goal**: cada campo de imagen que el método de pago ya muestra (`imageFields()`, hoy siempre el
QR) gana una opción de descarga que entrega el archivo como imagen local, con la misma
notificación no bloqueante de 5000 ms en éxito o error (FR-004, FR-006, FR-009, FR-010 y FR-011
para campos de imagen).

**Independent Test**: abrir "Datos de transferencia" de un método con un QR configurado, activar
la descarga, y confirmar que el archivo queda guardado en el dispositivo y coincide con el QR
mostrado; verificar la notificación de éxito de 5s (quickstart.md Escenario 2). No depende de US1.

### Tests for User Story 2

- [X] T011 [US2] En `transfer-details-step.component.spec.ts`: caso que, con un método construido
      vía `buildMethod` con un campo de imagen con valor, verifica que se renderiza un
      botón/ícono de descarga identificable junto al QR (FR-004, FR-010)
- [X] T012 [US2] En el mismo archivo: caso que mockea `fetch` global para que resuelva con una
      respuesta `ok` cuyo `.blob()` devuelve un `Blob` de prueba, mockea `URL.createObjectURL`/
      `URL.revokeObjectURL`, simula activar la descarga, y verifica que (a) `fetch` se llamó con la
      URL exacta del campo de imagen, (b) `URL.createObjectURL` se llamó con el blob devuelto, y
      (c) se llamó `toast.success(...)` con `5000` como segundo argumento (FR-004, FR-006,
      FR-007 para éxito, research.md D2-D3)
- [X] T013 [US2] En el mismo archivo: caso que mockea `fetch` para que rechace (o resuelva con
      `ok: false`), simula la misma activación, y verifica que se llamó `toast.error(...)` con
      `5000` como segundo argumento, que `toast.success` **no** se llamó, y que
      `URL.createObjectURL` **no** se llamó (FR-009, FR-007 para error)
- [X] T014 [US2] En el mismo archivo: caso que, con un método construido sin ningún campo de
      imagen con valor (solo campos de texto), verifica que no aparece ninguna opción de descarga
      en la pantalla (FR-011, segunda mitad)

### Implementation for User Story 2

- [X] T015 [US2] En `transfer-details-step.component.ts`: agregar
      `async downloadImage(url: string, filename: string): Promise<void>`, que hace
      `fetch(url)` dentro de un `try/catch`, valida `response.ok` (si no, lanza para caer en el
      `catch`), convierte a `Blob`, crea `const objectUrl = URL.createObjectURL(blob)`, dispara una
      `<a>` temporal (`a.href = objectUrl; a.download = filename; a.click()`), revoca la URL con
      `URL.revokeObjectURL(objectUrl)`, y notifica
      `this.toast.success('Imagen descargada', 5000)` en éxito o
      `this.toast.error('No se pudo descargar la imagen', 5000)` en el `catch` (research.md D2, D3,
      D6)
- [X] T016 [P] [US2] En `icon.component.ts`: agregar el caso `@case ('download')` con el trazo SVG
      estándar de "descargar" (flecha hacia abajo + bandeja), mismo estilo *single-stroke* que el
      resto del `@switch` (research.md D4)
- [X] T017 [US2] En `transfer-details-step.component.ts`: dentro del `@for (f of imageFields();
      track f.key)` (líneas ~62-71), agregar un botón junto al QR
      (`(click)="downloadImage(method()!.payment_info?.[f.key] ?? '', qrFilename(f))"`) con
      `<app-icon name="download">` y un `aria-label`/`title` de "Descargar"; agregar el método
      privado `qrFilename(f: PaymentMethodField): string` que arma `qr-<slug-del-método>.png`,
      agregando `f.label`/`f.key` al nombre solo si `imageFields().length > 1` (research.md D3,
      D7). Depende de T015, T016. Hace pasar T011, T012, T013, T014.
- [ ] T018 [US2] Ejecutar manualmente quickstart.md Escenario 2 (UI), incluyendo el paso de
      verificación de riesgo de CORS de R2 (research.md D2, quickstart.md paso 6)

**Checkpoint**: descargar la imagen del QR funciona de punta a punta, verificable de forma
independiente de US1.

---

## Phase 5: Polish & Cross-Cutting Concerns

- [X] T019 Ejecutar la suite completa de `transfer-details-step.component.spec.ts` (`ng test
      --watch=false --include='...transfer-details-step.component.spec.ts'`) y confirmar que
      todos los casos nuevos (T003-T006, T011-T014) pasan en verde — 8/8 en verde
- [X] T020 Ejecutar la suite completa de `pos-heladeria` (`ng test --watch=false`) y confirmar que
      los únicos fallos son los 4 ya documentados en T001, sin ningún fallo nuevo (Principio X) —
      581/585 en verde, mismos 4 fallos preexistentes (`app.spec.ts`, `auth.service.spec.ts`,
      `sidebar.component.spec.ts`, `pos-checkout-panel.component.spec.ts` T032; T001 solo había
      mostrado 3 de los 4 por el recorte de `tail -60`, el conteo total ya coincidía en 4)
- [ ] T021 Ejecutar los 2 escenarios de quickstart.md de punta a punta como validación final

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: sin dependencias.
- **Foundational (Phase 2)**: depende de Setup (T001). Bloquea todos los tests de US1 y US2 —
  ambos agregan casos al mismo archivo nuevo que T002 crea.
- **US1 (Phase 3)** y **US2 (Phase 4)**: dependen solo de Foundational (T002). Son funcionalmente
  independientes entre sí (spec.md), pero ambas agregan tests al mismo archivo
  (`transfer-details-step.component.spec.ts`) y métodos/template al mismo componente
  (`transfer-details-step.component.ts`) — coordinar el orden de merge si se trabajan a la vez.
- **Polish (Phase 5)**: depende de que ambas historias estén completas.

### Dentro de cada historia

- US1: T003-T006 (tests) antes de T007-T009 (implementación, hacen pasar los cuatro). T010 al
  final.
- US2: T011-T014 (tests) antes de T015-T017 (implementación, hacen pasar los cuatro). T018 al
  final.

### Parallel Opportunities

- T008 (`icon.component.ts`, caso `'copy'`) y T016 (`icon.component.ts`, caso `'download'`) son
  las únicas tareas de implementación marcadas `[P]` — cada una agrega un `@case` distinto al
  mismo archivo, pero no hay dependencia entre ambas ni con los tests de la otra historia.
- Una vez completa la Fase 2 (Foundational), los tests de US1 (T003-T006) y los de US2
  (T011-T014) pueden escribirse en paralelo por personas distintas si se coordina el mismo archivo
  al hacer merge — ambos bloques son casos nuevos sin solaparse.
- No se recomienda paralelismo real de *implementación* entre US1 y US2 sobre
  `transfer-details-step.component.ts` (T009 y T017 tocan el mismo `@for` distinto cada una, pero
  conviene revisarlas en el mismo orden que este documento para no pisarse el archivo).

---

## Parallel Example: Fase 3 + Fase 4 (íconos)

```bash
Task: "Agregar el caso 'copy' en icon.component.ts (T008)"
Task: "Agregar el caso 'download' en icon.component.ts (T016)"
```

---

## Implementation Strategy

### MVP First (Setup + Foundational + User Story 1)

1. Completar Phase 1 (Setup) — confirma el baseline (573/577 en verde, 4 fallos preexistentes
   ajenos).
2. Completar Phase 2 (Foundational) — harness del archivo de test nuevo.
3. Completar Phase 3 (US1) — entrega el ajuste de mayor prioridad (P1), el que previene un error
   real de transcripción del comensal.
4. **Detener y validar**: quickstart.md Escenario 1.
5. US2 se entrega después, de forma incremental, sin romper US1 (mismo archivo, regiones de
   template y de test distintas).

### Incremental Delivery

1. Setup + Foundational → harness listo, sin cambio de comportamiento observable todavía.
2. + US1 → copiar número de cuenta/celular → validar (MVP).
3. + US2 → descargar imagen del QR → validar (incluye la verificación de riesgo de CORS de R2,
   research.md D2).
4. + Polish.

---

## Notes

- Ningún task de este documento agrega dependencias nuevas ni toca `pos-backend` (spec.md,
  Assumptions; research.md D1, D2, D5).
- T012/T013 son el punto técnico más delicado de todo el plan: dependen de mockear `fetch`/`Blob`/
  `URL.createObjectURL` en el entorno de test (jsdom/Vitest) en vez de una llamada de red real —
  confirmar que el entorno de test del proyecto soporta esos mocks antes de darlas por
  imposibles de escribir; si algún mock no está disponible en el entorno actual, documentarlo como
  hallazgo nuevo antes de saltarlas.
- T018 es la única tarea de esta spec que puede descubrir el riesgo abierto de research.md D2 (CORS
  de R2) en el mundo real — si falla de forma consistente, no es un defecto de implementación de
  T015-T017 sino una configuración pendiente de infraestructura (ver quickstart.md paso 6).
- Commitear después de cada tarea o grupo lógico; detenerse en cada checkpoint para validar la
  historia de forma independiente antes de continuar con la siguiente.
