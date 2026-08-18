# Quickstart: validar la corrección del orden de borrado de imagen en R2 (A-44)

Guía de ejecución para comprobar que la corrección cumple su contrato. No repite firmas ni tablas
ya detalladas en [data-model.md](./data-model.md) y
[contracts/update-product-endpoint.md](./contracts/update-product-endpoint.md) — solo enlaza a
ellas.

**Prerequisitos**: entorno virtual de `pos-backend` activado (Python 3.14), ejecutado desde la raíz
de `../pos-backend` (sibling de este repo `pos-specs`).

```bash
cd ../pos-backend
source env/bin/activate
```

No hace falta PostgreSQL ni Cloudflare R2 reales: `app/characterization_tests/fixtures.py` crea
SQLite en memoria, y `delete_object` se mockea con `unittest.mock.patch` (research.md Decisión 2).

## Paso 0 — Confirmar que no existe hoy ningún characterization test de `products.service`

```bash
ls app/characterization_tests/ | grep products
```

**Resultado esperado antes de esta delta**: sin salida — no hay ningún `test_products_*.py`. Esta
delta crea el primero (`test_products_service.py`), acotado a `update_product` y A-44 (fuera de
alcance: caracterizar el resto de `ProductService`).

## Paso 1 — Escribir el characterization test que demuestra el defecto actual (antes de tocar el código)

Crear `app/characterization_tests/test_products_service.py` con un test que fuerza un fallo de
`db.commit()` y verifica el orden actual (`delete_object` antes del commit):

```python
"""CONGELA comportamiento actual → corregido: app/api/v1/products/service.py:78-91
(update_product) — A-44 (registro-de-anomalias.md, registro-riesgos.md R23).

Ejecutar solo este módulo:

    python -m unittest app.characterization_tests.test_products_service -v
"""
import unittest
from unittest import mock

from app.characterization_tests import fixtures as fx
from app.api.v1.products.service import ProductService
from app.api.v1.products.schemas import ProductUpdate

OLD_URL = "https://example.invalid/tenant/products/old.jpg"


class TestUpdateProductA44(unittest.TestCase):
    def _seed_product(self, db):
        # usar los helpers de fx.py para Category + Product con image_url=OLD_URL
        ...

    def test_a44_fallo_de_commit_no_deja_referencia_rota(self):
        """Tras la corrección: si el commit falla, el objeto viejo NO se borra
        (FR-002) — antes de la corrección, este mismo escenario borraba el
        objeto viejo pese al fallo (A-44)."""
        db = fx.new_session()
        product = self._seed_product(db)
        service = ProductService()

        with mock.patch(
            "app.api.v1.products.service.delete_object"
        ) as mock_delete, mock.patch.object(
            db, "commit", side_effect=RuntimeError("fallo ajeno a la imagen")
        ):
            with self.assertRaises(RuntimeError):
                service.update_product(
                    db, product.id, ProductUpdate(image_url="https://example.invalid/tenant/products/new.jpg")
                )

        mock_delete.assert_not_called()  # FR-002: nunca se llega a borrar

    def test_a44_camino_feliz_borra_despues_del_commit(self):
        """FR-001/FR-003: en éxito, delete_object se llama después de commit —
        mismo resultado final que antes de la corrección."""
        db = fx.new_session()
        product = self._seed_product(db)
        service = ProductService()
        orden = []

        with mock.patch(
            "app.api.v1.products.service.delete_object",
            side_effect=lambda key: orden.append("delete"),
        ), mock.patch.object(
            db, "commit", side_effect=lambda: orden.append("commit")
        ):
            service.update_product(
                db, product.id, ProductUpdate(image_url="https://example.invalid/tenant/products/new.jpg")
            )

        self.assertEqual(orden, ["commit", "delete"])  # orden nuevo, FR-001
```

```bash
python3 -m unittest app.characterization_tests.test_products_service -v
```

**Resultado esperado antes del fix**: `test_a44_fallo_de_commit_no_deja_referencia_rota` **falla**
(`mock_delete.assert_not_called()` no se cumple: el código actual borra antes del commit) —
confirma con datos reales el defecto que describe A-44. `test_a44_camino_feliz_borra_despues_del_commit`
también falla (`orden == ["delete", "commit"]`, orden invertido al esperado).

## Paso 2 — Aplicar la corrección

Editar `app/api/v1/products/service.py:78-89` según
[data-model.md](./data-model.md) y [research.md](./research.md) Decisión 1:

```python
# Antes (A-44):
if data.image_url is not None and data.image_url != product.image_url:
    old_image_url = product.image_url
    product.image_url = data.image_url
    if old_image_url:
        old_key = key_from_public_url(old_image_url)
        if old_key:
            delete_object(old_key)
if data.active is not None:
    product.active = data.active
if data.available is not None:
    product.available = data.available
db.commit()

# Después:
old_key = None
if data.image_url is not None and data.image_url != product.image_url:
    old_image_url = product.image_url
    product.image_url = data.image_url
    if old_image_url:
        old_key = key_from_public_url(old_image_url)
if data.active is not None:
    product.active = data.active
if data.available is not None:
    product.available = data.available
db.commit()
if old_key:
    delete_object(old_key)
```

Sin import nuevo — `delete_object`/`key_from_public_url` ya están importados en `service.py:16`.
Ningún otro cambio en la función: `create_product`, `soft_delete` y el resto de `update_product`
quedan intactos.

## Paso 3 — Confirmar la corrección

```bash
python3 -m unittest app.characterization_tests.test_products_service -v
```

**Resultado esperado tras el fix**: ambos tests en verde —

- `test_a44_fallo_de_commit_no_deja_referencia_rota`: `delete_object` nunca se llama cuando el
  commit falla (FR-002, CA2).
- `test_a44_camino_feliz_borra_despues_del_commit`: el orden observado es `["commit", "delete"]`
  (FR-001), y el resultado final (imagen vieja borrada, nueva persistida) es idéntico al de antes
  de la corrección (FR-003, CA1).

## Paso 4 — No regresión en el resto de `catalog`/`products`

```bash
python3 -m unittest app.characterization_tests.test_catalog_service_sku -v
python3 -m unittest app.characterization_tests.test_catalog_line_pricing -v
```

**Resultado esperado**: sin cambios — esta delta no toca `catalog_engine` ni el resto de
`ProductService`; estos módulos no referencian `delete_object`/`image_url`.

## Verificación final — SC-001 a SC-004

```bash
python3 -m unittest app.characterization_tests.test_products_service -v
```

Los dos tests en verde (contraste antes/después incluido en el propio Paso 1 vs. Paso 3, según
`FR-006`) es la señal de que la corrección está completa: ningún fallo de guardado deja ya una
referencia de imagen rota (CA2), el camino feliz no cambió (CA1), y no hay ningún mecanismo de
recálculo retroactivo agregado — `update_product` sigue operando producto por producto, sin tocar
ningún registro existente (CA3).
