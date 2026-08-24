# Contrato: Catálogo de Métodos de Pago (Super Admin)

Router nuevo `app/api/v1/super_admin/payment_methods_router.py`, montado bajo
`app/api/v1/super_admin/router.py` (hereda `dependencies=[Depends(get_current_super_admin)]` del
router padre — ningún endpoint de este contrato necesita repetir la dependencia). Cubre
**Historia de Usuario 1** (`spec.md`).

## `GET /super-admin/payment-methods-catalog`

Lista el catálogo completo (activos e inactivos — el Super Admin necesita ver ambos para
administrar).

**Response 200** — `list[PaymentMethodCatalogResponse]`:

```json
[
  {
    "id": "uuid",
    "name": "Nequi",
    "type": "transfer",
    "active": true,
    "fields": [
      {"key": "celular", "label": "Número de celular", "required": true, "format": "numeric", "length": 10},
      {"key": "qr", "label": "Código QR", "required": false, "format": "image"}
    ],
    "created_at": "2026-08-24T00:00:00Z",
    "updated_at": "2026-08-24T00:00:00Z"
  }
]
```

## `POST /super-admin/payment-methods-catalog`

Crea una entrada de catálogo (FR-001, RF-1). `name` único a nivel de plataforma → `409` si ya
existe (mismo patrón que `ensure_unique`).

**Request** — `PaymentMethodCatalogCreate`:

```json
{
  "name": "Daviplata",
  "type": "transfer",
  "fields": [
    {"key": "celular", "label": "Número de celular", "required": true, "format": "numeric", "length": 10}
  ]
}
```

`type` es obligatorio (a diferencia de `PaymentMethodCreate` hoy en `sales/schemas.py`, que lo
infiere de `is_cash`) — aquí no hay `is_cash` porque la clasificación completa (incluyendo "cash")
vive en `type` directamente; una sola entrada de catálogo puede tener `type="cash"` (Efectivo) sin
necesidad de un booleano paralelo.

**Response 201** — `PaymentMethodCatalogResponse` (mismo shape que el `GET`). `active=true` por
defecto (FR-001: "queda disponible para que los tenants lo activen").

**Errores**:
- `409` — `name` ya existe en el catálogo.
- `422` — `fields` con `key` duplicada, o `format` fuera de `{"text","numeric","image"}`, o
  `length` presente en un campo `format="image"`.

## `PATCH /super-admin/payment-methods-catalog/{id}`

Edita nombre/tipo/campos y/o activa-desactiva (FR-002/FR-003, RF-1).

**Request** — `PaymentMethodCatalogUpdate` (todos los campos opcionales):

```json
{ "fields": [ /* nueva lista completa de campos */ ], "active": false }
```

Editar `fields` **no** revalida ni cambia `is_complete` de configuraciones de tenant ya existentes
(edge case de `spec.md` — ver data-model.md, Decisión 4 de research.md). Desactivar (`active:
false`) no elimina ni desactiva las filas `tenant.payment_methods` que ya referencian este
catálogo — solo las saca del cálculo de disponibilidad de checkout (FR-013, ver
[checkout-payment-methods.md](./checkout-payment-methods.md)).

**Response 200** — `PaymentMethodCatalogResponse` actualizado.

**Errores**:
- `404` — no existe esa entrada de catálogo.
- `409` — el nuevo `name` colisiona con otra entrada.
- `422` — mismas validaciones de `fields` que en `POST`.

No existe `DELETE` — el catálogo no soporta borrado físico (Assumptions de `spec.md`).
