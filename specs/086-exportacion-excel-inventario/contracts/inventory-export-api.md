# Contrato de API: Exportación de Inventario a Excel

**Feature**: `086-exportacion-excel-inventario` | **Fecha**: 2026-09-22

Endpoint nuevo expuesto por `pos-backend` y consumido por `pos-heladeria`. Extiende el
router existente `app/api/v1/inventory/router.py` (prefijo `/inventory`, ya montado bajo
`/api/v1`).

## `GET /api/v1/inventory/items/export`

Genera y retorna un archivo `.xlsx` con el respaldo completo del inventario del tenant
autenticado (todos los insumos, activos e inactivos, sin filtros).

### Autenticación y autorización

- Requiere `Authorization: Bearer <token>` válido (igual que el resto de la API).
- Hereda `require_module_access("inventario")` del router (el plan del tenant debe incluir
  el módulo de Inventario).
- Requiere además `require_tenant_admin` (rol `ADMIN`) — ver `research.md` §3 para la
  justificación de por qué este endpoint es más estricto que `GET /inventory/items`.

### Request

Sin parámetros de query, sin body. La exportación **ignora** cualquier filtro, búsqueda u
orden — siempre es el total absoluto del inventario (FR-003).

```http
GET /api/v1/inventory/items/export HTTP/1.1
Authorization: Bearer <jwt>
```

### Response — éxito (200 OK)

```http
HTTP/1.1 200 OK
Content-Type: application/vnd.openxmlformats-officedocument.spreadsheetml.sheet
Content-Disposition: attachment; filename="inventario_2026-09-22.xlsx"
Content-Length: <bytes>

<contenido binario .xlsx>
```

**Estructura del archivo** (hoja única, sin nombre específico requerido por el spec):

| Fila | Nombre | Tipo | Unidad | Stock | Mínimo | Costo | Estado |
|------|--------|------|--------|-------|--------|-------|--------|
| 1 (encabezado) | `Nombre` | `Tipo` | `Unidad` | `Stock` | `Mínimo` | `Costo` | `Estado` |
| 2..N+1 | texto | texto | texto | número | número | número | texto |

- `N` = número total de insumos del tenant (activos + inactivos) en el momento de la
  petición.
- Con `N = 0`: la respuesta sigue siendo 200 con un `.xlsx` válido de una sola fila (el
  encabezado).
- Ver `data-model.md` para el detalle de cada columna y la fórmula exacta de "Estado".

### Response — errores

| Código | Cuándo | Body |
|--------|--------|------|
| `401 Unauthorized` | Token ausente, inválido o expirado (comportamiento estándar ya existente de la API, sin cambios) | JSON estándar de error de auth |
| `403 Forbidden` | Usuario autenticado pero sin rol `ADMIN`, o tenant sin el módulo `inventario` en su plan | `{"detail": "Tenant admin access required"}` (mismo mensaje que `require_tenant_admin` ya usa en `create_item`/`update_item`/`adjust_item`) o el mensaje estándar de `require_module_access` |
| `500 Internal Server Error` | Falla inesperada durante la construcción del archivo (p. ej. error de conexión a la base de datos a mitad del proceso) | JSON estándar de error de FastAPI. **Garantía**: nunca se envía un `.xlsx` parcial/corrupto — el workbook se construye completo en memoria antes de escribir cualquier byte de la respuesta (ver `research.md` §2) |

No hay un código de error específico para "inventario vacío" — ese caso es un 200 exitoso
con un archivo de solo encabezado (no es un error, ver Edge Cases del spec).

### Contrato de consumo desde el frontend (`pos-heladeria`)

`InventoryService` (`src/app/modules/inventory/services/inventory.service.ts`) agrega:

```typescript
exportItems(): Observable<HttpResponse<Blob>> {
  return this.http.get(`${this.baseUrl}/items/export`, {
    responseType: 'blob',
    observe: 'response',
  });
}
```

El componente (`inventory-page.component.ts`) es responsable de:
1. Invocar `exportItems()` al presionar el botón "Exportar Inventario".
2. En éxito: leer el nombre de archivo del encabezado `Content-Disposition` de la
   respuesta (o construir `inventario_<fecha-local>.xlsx` como respaldo si el encabezado no
   llega) y disparar la descarga vía blob URL + `<a download>` sintético (patrón de
   `research.md` §6).
3. En error: mostrar `this.toast.error(...)` con un mensaje entendible para el
   administrador (research.md §8) — sin dejar ningún archivo descargado.

No se introduce ningún nuevo endpoint, campo ni parámetro fuera de los descritos en este
documento.
