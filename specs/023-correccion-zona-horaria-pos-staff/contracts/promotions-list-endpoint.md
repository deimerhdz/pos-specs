# Contrato: `GET /promotions` (`list_promotions`)

`app/api/v1/promotions/router.py:37`. Endpoint existente, **sin cambios de forma** en esta spec —
mismo `response_model=Page[PromotionResponse]`, mismos parámetros de consulta, mismos códigos de
estado. Lo único que cambia es un header nuevo en la respuesta.

## Request

```
GET /promotions?status=active&size=100
Authorization: Bearer <access_token> (Depends(get_current_user)) — sin cambios
```

Sin cambios en los parámetros de consulta (`page`, `size`, `status`, `search`) ni en su validación.

## Response — `200 OK`

Cuerpo (`Page[PromotionResponse]`): **sin cambios**.

Headers: se agrega uno nuevo, listado en `expose_headers` de `CORSMiddleware`
(`app/main.py:95`) para que el JS del navegador pueda leerlo en un origen distinto
(`api.skeilopos.com` vs. el subdominio del tenant):

| Header | Valor | Ejemplo |
|---|---|---|
| `X-Server-Time` | ISO 8601, UTC, `datetime.now(timezone.utc).isoformat()` | `2026-08-18T22:30:05.123456+00:00` |

No es la hora local del tenant (`TENANT_TIMEZONE`) — es UTC. La conversión a hora local la sigue
haciendo, sin cambios, `promotion-pricing.util.ts` (`isPromoActiveNow`/`inTimeWindow`, que ya operan
sobre un `Date` de JavaScript, que en sí mismo no tiene zona horaria — internamente siempre es un
instante UTC; el defecto de A-09 nunca fue "el cliente no sabe convertir a hora local", fue "el
cliente arrancaba desde el instante equivocado"). `PromotionService.now()` corrige el instante de
partida; el resto del cálculo de vigencia no cambia.

## Consumidor (`pos-heladeria`)

Solo `PromotionService.activeQuery` (la query `['promotions', 'active']`, usada exclusivamente por
`PosTerminalStore`) lee este header. `pageQuery` (tabla de administración) y `candidatesQuery`
(detección de solapamientos) siguen ignorándolo — no lo necesitan porque no evalúan vigencia contra
un reloj (ver spec.md, Out of Scope: el panel de administración queda fuera de esta corrección).

## Compatibilidad

Header aditivo y opcional de leer: un cliente que no lo lea (versión de frontend anterior a esta
spec, o cualquier otro consumidor de la API) sigue funcionando exactamente igual que hoy — no hay
cambio de comportamiento observable para nadie que no sea `PromotionService.activeQuery`.
