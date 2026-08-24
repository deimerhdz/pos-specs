# Contrato: `GET /tenant` y `PATCH /tenant`

`app/api/v1/tenant/router.py` (`TenantInfoResponse`, `tenant/schemas.py`). Endpoints existentes.
`GET` gana un campo nuevo de solo lectura; `PATCH` **no** lo acepta — decisión explícita de
Clarifications (spec.md): la zona horaria del tenant no se configura por autoservicio en esta spec.

## `GET /tenant` — gana `timezone`

**Antes**:

```json
{
  "id": 7,
  "name": "Heladería Central",
  "host": "central.skeilopos.com",
  "plan": "basic",
  "logo_url": null,
  "receipt_message": null,
  "invoice_prefix": "FC"
}
```

**Después**:

```json
{
  "id": 7,
  "name": "Heladería Central",
  "host": "central.skeilopos.com",
  "plan": "basic",
  "logo_url": null,
  "receipt_message": null,
  "invoice_prefix": "FC",
  "timezone": "America/Bogota"
}
```

`timezone` siempre presente y siempre un nombre IANA válido (garantizado por el `server_default` de la
migración + el validador `@validates` del modelo — nunca `null`, nunca una cadena inválida). Consumido
por `TenantInfoService.info()` en el frontend, fuente de zona horaria para `TenantDatePipe` y
`businessToday()` (research.md Decisiones 6-7).

## `PATCH /tenant` — `timezone` explícitamente fuera de `TenantUpdateRequest`

`TenantUpdateRequest` sigue aceptando exactamente los mismos campos que hoy (`name`, `logo_url`,
`receipt_message`, y los demás ya editables) — **`timezone` no se agrega a este schema**. Un intento
de incluir `"timezone": "..."` en el cuerpo de un `PATCH /tenant` es ignorado (campo no declarado en
el schema de request, comportamiento estándar de Pydantic con un modelo que no lo define) exactamente
igual que cualquier otro campo no reconocido lo sería hoy.

**Único camino de escritura**: `app/scripts/set_tenant_timezone.py` (research.md Decisión 4), o el
valor por defecto de la migración. Ninguna ruta HTTP permite cambiarlo.

## Consumidor (`pos-heladeria`)

`TenantInfoService.info()` (signal, `tenant-info.service.ts`) — la interfaz `TenantInfo` gana
`timezone: string`, poblada por `load()` (`GET /tenant`, sin cambios de wiring, solo de forma de la
respuesta). `update()` (`PATCH /tenant`) sigue tipado sobre `Partial<TenantInfo>` a nivel de
TypeScript, pero como el backend ignora `timezone` si llegara a incluirse, no hay forma de que un
cambio accidental en el frontend lo persista — la garantía real vive en el backend
(`TenantUpdateRequest`), no en la disciplina del código cliente.

## Compatibilidad

Campo aditivo en `GET /tenant` — cualquier cliente que no lo lea (o cualquier otra versión de
frontend) sigue funcionando exactamente igual que hoy. `PATCH /tenant` no cambia su contrato de
request en absoluto.
