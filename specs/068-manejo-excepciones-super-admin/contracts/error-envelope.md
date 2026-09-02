# Contrato: respuesta de error del módulo super-admin

Aplica a toda ruta bajo el prefijo `/api/v1/super-admin` (incluye `/super-admin/plans/**` y
`/super-admin/payment-methods-catalog/**`, montados bajo el mismo router). No aplica a ningún
otro módulo del backend.

## Forma general

Toda respuesta con código de estado 4xx o 5xx de este módulo tiene este cuerpo (ver
`data-model.md` § 2 para la tabla de campos y ejemplos completos):

```json
{
  "success": false,
  "error": {
    "code": "STRING_ESTABLE",
    "message": "Texto legible y seguro",
    "details": null
  },
  "request_id": "uuid-v4",
  "detail": "Texto legible y seguro (igual a error.message, por compatibilidad)"
}
```

## Mapeo código HTTP ↔ `error.code` por defecto

Ver tabla completa en `research.md` § 8. Resumen:

| HTTP | `error.code` por defecto |
|---|---|
| 401 | `UNAUTHORIZED` |
| 403 | `FORBIDDEN` |
| 404 | `NOT_FOUND` (o un código específico de entidad si la excepción de dominio lo declara, p. ej. `TENANT_NOT_FOUND`) |
| 409 | `CONFLICT` |
| 422 | `INVALID_INPUT` |
| 500 | `INTERNAL_ERROR` |

## Garantías

1. **Nunca** aparece en `error.message`/`detail`/`error.details`: fragmentos de SQL, trazas de
   pila, nombres de esquema de base de datos, rutas de archivo del servidor, credenciales,
   tokens (FR-003, SC-002).
2. Para 401/403, el cuerpo **nunca** confirma ni niega la existencia de un tenant, usuario o plan
   específico (FR-004).
3. `request_id` está presente en el 100% de las respuestas de error de este módulo, y coincide
   con el identificador correlacionable en los logs del servidor y, en producción, en el evento
   de Sentry correspondiente cuando el error es una falla técnica inesperada (FR-015, SC-003).
4. Endpoints exitosos (2xx) **no** cambian de forma — este contrato aplica únicamente a
   respuestas de error.

## Casos fuera de este contrato

- Errores de validación de esquema de FastAPI/Pydantic (422) mantienen, además de este envelope,
  el cuerpo de validación estándar que Pydantic ya produce hoy — este spec no lo reemplaza, solo
  lo envuelve en los campos de nivel superior (`success`, `error.code = "INVALID_INPUT"`,
  `request_id`) para que siga siendo consistente con el resto del contrato.
- Cualquier ruta fuera de `/api/v1/super-admin` sigue devolviendo exactamente lo que devuelve
  hoy (`{"detail": "..."}` plano) — no está cubierta por este contrato ni por el middleware nuevo.
