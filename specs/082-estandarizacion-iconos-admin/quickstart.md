# Quickstart — Validación: Estandarización de Íconos en el Panel de Administración

**Spec**: [spec.md](./spec.md) | **Plan**: [plan.md](./plan.md) | **Catálogo**: [data-model.md](./data-model.md)

Guía para validar, de punta a punta, que la implementación cumple los criterios de éxito de
`spec.md`. No sustituye los tests automatizados de `tasks.md` — es la validación manual/exploratoria
de la funcionalidad completa.

## Prerrequisitos

- Repositorio `pos-heladeria` con la dependencia `material-icons` instalada y los archivos de fuente
  Outlined copiados a `public/fonts/material-icons/` (research.md, Decisión D1).
- Build de desarrollo levantado: `ng serve` (o el comando equivalente del proyecto) sirviendo el
  panel de administración con un usuario autenticado (cualquier rol con acceso al panel principal).

## Escenario 1 — Consistencia visual (US1, SC-001)

1. Iniciar sesión y navegar, en orden: panel principal → caja → inventario → mesas/terminal →
   usuarios → configuración → promociones.
2. En cada pantalla, verificar visualmente que todos los íconos (navegación, botones de acción,
   indicadores de estado, tarjetas de estadísticas) comparten el mismo estilo de trazo (Material
   Icons Outlined) — ninguno debe verse como un emoji ni como un SVG de trazo distinto.
3. **Resultado esperado**: cero emoji y cero SVG artesanales visibles en las siete pantallas.

## Escenario 2 — Sin temática de heladería (US2, SC-003)

1. Cerrar sesión y volver a entrar sin un logo de tenant configurado (o usar un tenant de prueba sin
   logo propio).
2. Verificar que el panel lateral muestra un ícono de tienda/negocio genérico, no un cono de helado.
3. Ir al panel principal y verificar que la tarjeta/acceso "Productos" usa un ícono neutro (bolsa de
   compras), no un cono de helado.
4. **Resultado esperado**: ninguna búsqueda visual en las siete pantallas encuentra un ícono con
   temática de heladería.

## Escenario 3 — Componente único y reutilizable (US3)

1. Revisar el código fuente de las siete pantallas del panel de administración y confirmar que solo
   un componente de ícono se importa/usa en todas ellas (el nuevo, no `app-icon`).
2. Confirmar que `src/app/shared/icon/icon.component.ts` (`app-icon`) sigue existiendo sin cambios en
   el repositorio, y que sus únicos consumidores restantes son archivos del flujo público de menú QR
   (`public-menu.component.ts`, `checkout/*-step.component.ts`) — ver research.md, Decisión D2.
3. **Resultado esperado**: ninguna pantalla del panel de administración importa `app-icon`.

## Escenario 4 — Accesibilidad (FR-011, SC-006)

1. Con una herramienta de auditoría de accesibilidad (p. ej. axe DevTools) o un lector de pantalla,
   recorrer cada botón de solo ícono y cada indicador de estado sin etiqueta visible en las siete
   pantallas.
2. **Resultado esperado**: cada uno anuncia una descripción equivalente a su significado (por el
   `aria-label` del botón contenedor o por el `ariaLabel` del ícono, según `contracts/icon-component-contract.md`); ninguno queda mudo para el lector de pantalla.

## Escenario 5 — Funciona sin conexión (FR-001, SC-007)

1. Cargar el panel de administración una vez con conexión a internet (para que el service worker
   cachee los assets, incluida la fuente de Material Icons).
2. Desconectar la red (modo avión o bloquear el dominio) y recargar la aplicación.
3. Navegar las siete pantallas del panel de administración.
4. **Resultado esperado**: todos los íconos se siguen mostrando correctamente; ninguno aparece como
   un cuadro vacío o un carácter faltante.

## Escenario 6 — El flujo público no cambia de comportamiento (FR-010, regresión)

1. Sin iniciar sesión, escanear/abrir el enlace del menú digital de una mesa de prueba
   (`/menu/t/:token`).
2. Navegar el menú, agregar productos al carrito y avanzar hasta el paso de checkout.
3. **Resultado esperado**: el flujo completo funciona exactamente igual que antes de esta
   funcionalidad (mismas rutas, mismas reglas, mismo resultado final). Los íconos de los componentes
   compartidos (`cart.component.ts`, `product-select.component.ts`,
   `payment-attempt-review-panel.component.ts`, `pos-catalog-drawer.component.ts`) pueden verse
   visualmente distintos (research.md, Decisión D6) — eso es esperado, no un defecto; lo que se
   valida aquí es que ninguna regla de negocio ni navegación del flujo público cambió.
