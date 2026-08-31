# Implementation Plan: Copiar número de cuenta y descargar QR en pagos por transferencia

**Branch**: `060-copiar-cuenta-descargar-qr` | **Date**: 2026-08-30 | **Spec**: [spec.md](./spec.md)

**Input**: Feature specification from `/specs/060-copiar-cuenta-descargar-qr/spec.md`

## Summary

Dos mejoras de interacción sobre el paso "Datos de transferencia" del checkout del comensal
(`TransferDetailsStepComponent`, único punto del sistema que hoy muestra los datos de transferencia
de un método de pago distinto a efectivo — spec.md, Assumptions): (1) cada campo de texto ya
renderizado (`textFields()`) gana un ícono de copiar que escribe su valor al portapapeles vía
`navigator.clipboard.writeText()` (research.md D1); (2) cada campo de imagen ya renderizado
(`imageFields()`, hoy siempre el QR) gana una opción de descarga que trae el archivo con `fetch` +
`Blob` y lo entrega como archivo local vía una ancla temporal (research.md D2-D3) — no basta con el
patrón de descarga directa que ya usa `TableQrComponent`, porque esa imagen es una data URL local y
esta es una URL remota de Cloudflare R2 (cross-origin). Ambas acciones notifican con un toast no
modal de 5000 ms exactos, éxito o error (research.md D6), reutilizando `ToastService` ya existente.
Se agregan dos íconos nuevos (`'copy'`, `'download'`) al `IconComponent` compartido, mismo estilo
*single-stroke* que el resto del set (research.md D4). Ningún cambio de modelo de datos, de
contrato de backend, ni al flujo de envío del pedido/comprobante (spec.md FR-012).

**Riesgo abierto para la fase de implementación** (research.md D2): no se puede confirmar por
código estático si el bucket de R2 permite `fetch()` cross-origin (CORS de lectura) sobre las URLs
públicas de `payment-methods` — la configuración vive del lado de Cloudflare. Queda como
verificación obligatoria (Principio X) antes de dar la Historia 2 por completada.

## Technical Context

**Language/Version**: TypeScript 5.9.2, Angular 21.1 (componentes standalone, signals, control flow
`@if`/`@for`, ya en uso por `TransferDetailsStepComponent`).

**Primary Dependencies**: ninguna nueva — Clipboard API y Fetch API nativas del navegador
(research.md D1-D2); `@angular/core` ya en uso.

**Storage**: N/A — sin cambios de base de datos ni de modelo de datos (spec.md no define "Key
Entities"; ambas mejoras operan sobre datos que el componente ya recibe hidratados).

**Testing**: Vitest vía `@angular/build:unit-test`. No existe hoy ningún `*.component.spec.ts` bajo
`src/app/modules/tables/pages/checkout/` (research.md D5) — este feature agrega el primero para
`TransferDetailsStepComponent`.

**Target Platform**: Web — SPA Angular del flujo de pedido del comensal por QR
(`/menu/t/:token/checkout/...`), mayormente navegadores móviles (el comensal escanea el QR con su
celular). Sin cambios en `../pos-backend` ni en ningún contrato de endpoint.

**Project Type**: cambio acotado al repositorio hermano `../pos-heladeria` únicamente — no toca
`../pos-backend` ni este repositorio de specs (`pos-specs`).

**Performance Goals**: sin objetivos numéricos nuevos.

**Constraints**: la descarga del QR depende de que el bucket de Cloudflare R2 permita lectura
cross-origin (`fetch`) sobre sus URLs públicas de `payment-methods` — riesgo identificado en
research.md D2, sin poder confirmarse por código estático, pendiente de verificación en
implementación; el valor copiado al portapapeles debe coincidir exactamente con el texto mostrado,
sin transformaciones (spec.md FR-003); ambas notificaciones (éxito y error) deben durar exactamente
5000 ms (spec.md FR-007, research.md D6).

**Scale/Scope**: 2 archivos de producción (`transfer-details-step.component.ts`:
2 métodos nuevos + ajustes de template; `icon.component.ts`: 2 casos `@switch` nuevos), 1 archivo de
test nuevo (`transfer-details-step.component.spec.ts`, sin tests existentes que ajustar en este
directorio — research.md D5).

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

- **Principio I (Nace de un spec)** ✅ — `specs/060-copiar-cuenta-descargar-qr/spec.md` existe, sin
  `[NEEDS CLARIFICATION]` pendiente (checklist de calidad 16/16, `/speckit-clarify` confirmó sin
  ambigüedades críticas).
- **Principio II (Comportamiento existente protegido)** ✅ — ambas mejoras son aditivas: agregan una
  acción nueva (copiar, descargar) sobre datos que la pantalla ya muestra hoy; no retiran ni
  cambian ningún comportamiento existente de `TransferDetailsStepComponent` (subir comprobante,
  enviar pedido, navegación atrás/salir siguen intactos — spec.md FR-012). No aplica registrar
  ninguna decisión en `registro-de-anomalias.md`: no hay comportamiento previo que cambie, solo
  capacidad nueva.
- **Principio III (Characterization tests)** — no aplica: no existe ningún test hoy sobre este
  componente (research.md D5), así que no hay ningún test con prefijo `"CONGELA comportamiento
  actual:"` que pueda verse afectado.
- **Principio IV (Nuevo comportamiento vía spec)** ✅ — el comportamiento nuevo está definido en
  spec.md, FR-001 a FR-012.
- **Principio V (No refactors oportunistas)** ✅ — no se toca ningún archivo fuera de
  `transfer-details-step.component.ts` y `icon.component.ts` (research.md D5); no se extrae ningún
  helper compartido de copiar/descargar a un servicio, porque ninguna otra pantalla lo necesita hoy
  (spec.md Assumptions); no se cambia el valor por defecto de `ToastService.success()` para el
  resto de la app (research.md D6), solo se pasa `5000` explícito en los call sites nuevos.
- **Principio VI (Evolución incremental)** ✅ — dos historias independientes, acotadas a un único
  componente y al sistema de íconos ya compartido; sin mezclar con ninguna refactorización,
  migración de datos ni cambio de arquitectura.
- **Principio VII (Datos históricos)** ✅ — no aplica; ninguna `Sale`/factura se toca, la pantalla
  afectada es previa al envío del pedido.
- **Principio VIII (Evolución del modelo de datos)** ✅ — no aplica; sin cambios de esquema.
- **Principio IX (Dependencias nuevas)** ✅ — ninguna dependencia nueva; Clipboard API y Fetch API
  son nativas del navegador (research.md D1-D2), justificado explícitamente frente a alternativas
  de librería.
- **Principio X (Verificación obligatoria)** — pendiente de ejecución en `/speckit-tasks` +
  `/speckit-implement`: tests nuevos para `copyField()` (éxito/fallo de Clipboard API) y
  `downloadImage()` (éxito/fallo de `fetch`), más la verificación manual contra un QR real
  documentada como riesgo abierto (research.md D2, Constraints arriba) antes de cerrar la Historia 2.
- **Principio XI (Negocio vs. técnico)** ✅ — la necesidad de negocio (copiar número, descargar QR,
  notificación no modal de 5s) viene del dueño/desarrollador en spec.md; las decisiones de este plan
  (research.md D1-D7) son todas técnicas — en particular, el mecanismo `fetch` + blob para la
  descarga es una decisión técnica para lograr el resultado de negocio ya pedido, no una
  funcionalidad nueva no solicitada.
- **Principio XII (Trazabilidad)** ✅ — Necesidad (pedida directamente por el dueño/desarrollador,
  ver spec.md Input) → Spec 060 → este Plan (research.md D1-D7) → Tasks/Implementación/Tests
  (siguientes comandos).
- **Principio XIII (Español de Colombia)** ✅ — todo este documento y los artefactos generados se
  redactan en español de Colombia.

**Resultado**: Gate PASA. Sin desviaciones que requieran justificación adicional a la ya
documentada en research.md D2 (riesgo de CORS, no una violación de principio — es una verificación
pendiente, no una decisión que contradiga la Constitución). Sin Complexity Tracking.

**Re-chequeo post-diseño (tras Fase 1)**: no se generó `data-model.md` ni `contracts/` — spec.md no
tiene sección "Key Entities" (sin datos nuevos involucrados) y ningún endpoint ni contrato de API
cambia de forma (ambas acciones son 100% del lado del cliente: Clipboard API y Fetch API sobre una
URL que el frontend ya recibe). `quickstart.md` cubre la validación manual completa de ambas
historias de usuario, incluyendo el riesgo de CORS de R2. Gate sigue PASANDO.

## Project Structure

### Documentation (this feature)

```text
specs/060-copiar-cuenta-descargar-qr/
├── plan.md                        # Este archivo (/speckit-plan)
├── research.md                    # Fase 0 (/speckit-plan) — decisiones D1-D7
├── quickstart.md                  # Fase 1 (/speckit-plan) — 2 escenarios de validación manual
├── checklists/requirements.md     # /speckit-specify — checklist de calidad de la spec
└── tasks.md                       # Fase 2 (/speckit-tasks) — aún no generado
```

Sin `data-model.md` ni `contracts/` — no aplican a esta spec (ver Constitution Check, re-chequeo
post-diseño).

### Source Code (repositorio de la aplicación)

El código vive en el repositorio hermano `../pos-heladeria` (Angular), no en este repositorio de
specs (`pos-specs`). Sin cambios en `../pos-backend`.

```text
pos-heladeria/
├── src/app/modules/tables/pages/checkout/
│   ├── transfer-details-step.component.ts       # copyField()/downloadImage() nuevos,
│   │                                               íconos + notificaciones en el template
│   │                                               (research.md D1-D3, D6-D7)
│   └── transfer-details-step.component.spec.ts  # nuevo — primer test de este componente
│                                                   (research.md D5)
└── src/app/shared/icon/
    └── icon.component.ts                        # casos 'copy' y 'download' nuevos en el
                                                     @switch (research.md D4)
```

**Structure Decision**: sin componentes ni servicios nuevos — las dos historias caen dentro de un
único archivo de producción ya existente (`transfer-details-step.component.ts`) más el sistema de
íconos ya compartido por toda la app (`icon.component.ts`); no se toca `checkout-progress.store.ts`
ni `diner.interface.ts` (ni `payment_info` ni `fields` cambian de forma — research.md D5, D7).

## Complexity Tracking

*Sin violaciones que justificar — ver "Resultado" en Constitution Check.*
