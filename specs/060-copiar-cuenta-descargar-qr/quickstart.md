# Quickstart de validación: Copiar número de cuenta y descargar QR en pagos por transferencia

**Spec**: [spec.md](./spec.md) | **Plan**: [plan.md](./plan.md)

Prerrequisitos: al menos un método de pago distinto a efectivo activo, con un campo de texto (p.
ej. "Número de celular") y un campo de imagen QR ya configurados (`payment-methods-page.component.ts`
del lado admin); un carrito de comensal con ítems, accedido vía el enlace/QR de una mesa
(`/menu/t/:token`), navegando hasta elegir ese método de pago en el paso 2 del checkout.

## Escenario 1 — Historia 1: copiar el número de cuenta/celular (FR-001/FR-002/FR-003/FR-008)

1. Elegir el método de pago no-efectivo configurado y llegar a "Paso 3 de 3: Datos de
   transferencia".
2. Verificar que junto al valor de texto (p. ej. "Número de celular: 3106448749") aparece un ícono
   reconocible de copiar.
3. Tocar/hacer clic en el ícono de copiar.
4. Pegar (Ctrl+V / long-press "Pegar") en otro campo de texto (p. ej. la barra de direcciones u
   otro input) y verificar que el valor pegado coincide exactamente con `3106448749`, sin espacios
   ni caracteres adicionales.
5. Verificar que aparece una notificación no bloqueante confirmando la copia, y que desaparece por
   sí sola a los 5 segundos sin haberla cerrado manualmente.
6. (Si el navegador lo permite simular) denegar el permiso de portapapeles y repetir el paso 3:
   verificar que aparece una notificación de error no bloqueante en su lugar, también no modal.
7. Si el método configurado tiene más de un campo de texto, repetir los pasos 2-5 para cada uno de
   forma independiente.

**Resultado esperado**: el comensal copia el valor exacto con una sola acción, sin seleccionar
texto manualmente, y ve una confirmación efímera no bloqueante.

## Escenario 2 — Historia 2: descargar la imagen del QR (FR-004/FR-006/FR-009)

1. En la misma pantalla, verificar que junto al código QR aparece una opción clara de descarga.
2. Activar la descarga.
3. Verificar que el navegador guarda un archivo de imagen en el dispositivo (o en la carpeta de
   descargas del sistema operativo) — no una nueva pestaña mostrando la imagen, ni un intento
   fallido silencioso.
4. Abrir el archivo descargado fuera de la app (galería/explorador de archivos) y verificar que
   corresponde exactamente al QR mostrado en pantalla.
5. Verificar que aparece una notificación no bloqueante confirmando la descarga, que desaparece por
   sí sola a los 5 segundos.
6. **Verificación de riesgo (research.md D2)**: si el paso 2 falla de forma consistente (la imagen
   nunca se descarga, sin importar el navegador probado), revisar primero si el error es de red o
   de CORS del bucket de Cloudflare R2 (inspeccionar la consola del navegador: un error de
   `fetch`/CORS ahí confirma que es una configuración pendiente del lado de infraestructura, no un
   defecto del componente) antes de dar la Historia 2 por completada; verificar en ese caso que de
   todos modos aparece la notificación de error no bloqueante (FR-009), en vez de fallar en
   silencio.

**Resultado esperado**: el comensal obtiene el archivo de imagen del QR sin necesidad de tomar una
captura de pantalla, con una confirmación efímera no bloqueante.
