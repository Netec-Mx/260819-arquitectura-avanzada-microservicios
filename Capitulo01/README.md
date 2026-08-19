# 4 Laboratorio: prototipo de servicio con FastAPI y pruebas contractuales

## Metadatos

| Campo | Valor |
|-------|-------|
| **Duración** | 42 minutos |
| **Complejidad** | Media |
| **Nivel Bloom** | Crear |

## Descripción General

En este laboratorio fundacional construirás desde cero dos microservicios — **OrderService** (puerto 8001) e **InventoryService** (puerto 8002) — aplicando los principios de bounded context estudiados en la lección teórica. Seguirás un enfoque **API-first**: diseñarás los contratos OpenAPI 3.1 antes de implementar el código, modelarás las entidades de dominio con Pydantic v2 y validarás la conformidad de la implementación mediante pruebas contractuales automatizadas con schemathesis. La estructura de proyecto generada será el workspace permanente para los laboratorios siguientes del curso.

## Objetivos de Aprendizaje

- [ ] Definir bounded contexts para los dominios de Pedidos e Inventario, identificando entidades y sus fronteras
- [ ] Implementar modelos de dominio con Pydantic v2 y endpoints RESTful conformes al contrato OpenAPI
- [ ] Estructurar el proyecto Python con separación de capas (routers, services, schemas, models)
- [ ] Ejecutar pruebas contractuales automatizadas con schemathesis validando conformidad con el contrato OpenAPI
- [ ] Dockerizar ambos servicios con un docker-compose.yml funcional

## Prerrequisitos

### Conocimientos

- Fundamentos de FastAPI y Pydantic (creación de modelos, validación)
- Principios REST y códigos de respuesta HTTP
- Uso básico de Docker y Docker Compose
- Comprensión de bounded contexts (lección 1.1)

### Acceso y herramientas

- Python 3.12.3 instalado vía pyenv 2.4.1
- Docker Engine 26.1.3 y Docker Compose 2.27.1
- Acceso a internet para descargar paquetes pip
- Editor de código (VS Code recomendado)

## Entorno del Laboratorio

### Software requerido

| Herramienta | Versión | Verificación |
|-------------|---------|--------------|
| Python | 3.12.3 | `python --version` |
| pip | ≥ 24.0 | `pip --version` |
| Docker Engine | 26.1.3 | `docker --version` |
| Docker Compose | 2.27.1 | `docker compose version` |

### Preparación inicial del workspace

```bash
# Crear la estructura raíz del proyecto
mkdir -p ~/microservices-lab/{order-service,inventory-service,shared,scripts,docs,tests}
cd ~/microservices-lab
```

---

## Paso 1: Análisis de Bounded Contexts y diseño de entidades

**Objetivo:** Identificar los dominios de Pedidos e Inventario como bounded contexts separados, definir sus entidades principales y documentar las fronteras.

### Instrucciones

1. Crea el documento de diseño que formaliza los bounded contexts:

```bash
cat > ~/microservices-lab/docs/bounded-contexts.md << 'EOF'
# Bounded Contexts — Microservices Lab

## Contexto: Pedidos (Orders)
- **Responsabilidad:** Gestión del ciclo de vida de pedidos
- **Entidades principales:**
  - `Order`: id, customer_id, status, items, created_at
  - `OrderItem`: product_id, quantity, unit_price
- **Estados válidos:** pending → confirmed → shipped → delivered | cancelled
- **Puerto:** 8001

## Contexto: Inventario (Inventory)
- **Responsabilidad:** Control de stock y reservas de producto
- **Entidades principales:**
  - `Product`: id, name, sku, stock_quantity, reorder_threshold
  - `StockReservation`: id, product_id, order_id, quantity, status
- **Estados de reserva:** pending → confirmed → released
- **Puerto:** 8002

## Fronteras de integración
- OrderService referencia productos SOLO por `product_id` (no importa el modelo completo)
- InventoryService referencia pedidos SOLO por `order_id`
- Comunicación futura vía eventos Kafka (Lab 02)
EOF
```

2. Verifica que la documentación se creó correctamente:

```bash
cat ~/microservices-lab/docs/bounded-contexts.md
```

### Resultado esperado

El archivo muestra los dos contextos con sus entidades, estados y regla de integración: cada contexto solo referencia al otro mediante IDs externos.

### Verificación

```bash
test -f ~/microservices-lab/docs/bounded-contexts.md && echo "✅ Documento de bounded contexts creado"
```

---

## Paso 2: Estructura del proyecto y entornos virtuales

**Objetivo:** Crear la estructura de directorios modular con separación de capas para ambos servicios y configurar entornos virtuales independientes.

### Instrucciones

1. Crea la estructura de directorios para **OrderService**:

```bash
mkdir -p ~/microservices-lab/order-service/{app/{routers,services,schemas,models},tests}
touch ~/microservices-lab/order-service/app/__init__.py
touch ~/microservices-lab/order-service/app/routers/__init__.py
touch ~/microservices-lab/order-service/app/services/__init__.py
touch ~/microservices-lab/order-service/app/schemas/__init__.py
touch ~/microservices-lab/order-service/app/models/__init__.py
```

2. Crea la estructura de directorios para **InventoryService**:

```bash
mkdir -p ~/microservices-lab/inventory-service/{app/{routers,services,schemas,models},tests}
touch ~/microservices-lab/inventory-service/app/__init__.py
touch ~/microservices-lab/inventory-service/app/routers/__init__.py
touch ~/microservices-lab/inventory-service/app/services/__init__.py
touch ~/microservices-lab/inventory-service/app/schemas/__init__.py
touch ~/microservices-lab/inventory-service/app/models/__init__.py
```

3. Crea el entorno virtual y dependencias para **OrderService**:

```bash
cd ~/microservices-lab/order-service
python -m venv .venv
source .venv/bin/activate

cat > requirements.txt << 'EOF'
fastapi==0.111.0
uvicorn==0.29.0
pydantic==2.7.1
httpx==0.27.0
pytest==8.2.1
pytest-asyncio==0.23.7
schemathesis==3.33.4
EOF

pip install -r requirements.txt
deactivate
```

4. Crea el entorno virtual y dependencias para **InventoryService**:

```bash
cd ~/microservices-lab/inventory-service
python -m venv .venv
source .venv/bin/activate

cat > requirements.txt << 'EOF'
fastapi==0.111.0
uvicorn==0.29.0
pydantic==2.7.1
httpx==0.27.0
pytest==8.2.1
pytest-asyncio==0.23.7
schemathesis==3.33.4
EOF

pip install -r requirements.txt
deactivate
```

### Resultado esperado

Dos servicios con estructura idéntica de capas y entornos virtuales independientes con todas las dependencias instaladas.

### Verificación

```bash
echo "--- Estructura OrderService ---"
find ~/microservices-lab/order-service/app -type f | sort
echo ""
echo "--- Estructura InventoryService ---"
find ~/microservices-lab/inventory-service/app -type f | sort
echo ""
# Verificar instalación de paquetes
source ~/microservices-lab/order-service/.venv/bin/activate
python -c "import fastapi; import pydantic; print(f'FastAPI {fastapi.__version__}, Pydantic {pydantic.__version__}')"
deactivate
```

Salida esperada:
```
FastAPI 0.111.0, Pydantic 2.7.1
```

---

## Paso 3: Implementación de schemas Pydantic v2 (modelos de dominio)

**Objetivo:** Definir los modelos de dominio como schemas Pydantic v2, respetando los bounded contexts — cada servicio tiene su propia representación de las entidades.

### Instrucciones

1. Crea los schemas del **OrderService**:

```bash
cat > ~/microservices-lab/order-service/app/schemas/order.py << 'EOF'
"""Schemas del bounded context de Pedidos."""
from datetime import datetime
from enum import Enum
from uuid import UUID, uuid4

from pydantic import BaseModel, Field


class OrderStatus(str, Enum):
    PENDING = "pending"
    CONFIRMED = "confirmed"
    SHIPPED = "shipped"
    DELIVERED = "delivered"
    CANCELLED = "cancelled"


class OrderItemCreate(BaseModel):
    """Datos necesarios para crear un item dentro de un pedido."""
    product_id: UUID
    quantity: int = Field(gt=0, description="Cantidad debe ser mayor a 0")
    unit_price: float = Field(gt=0, description="Precio unitario en EUR")


class OrderItemResponse(OrderItemCreate):
    """Representación de un item en la respuesta."""
    subtotal: float = Field(description="quantity * unit_price")


class OrderCreate(BaseModel):
    """Payload para crear un nuevo pedido."""
    customer_id: UUID
    items: list[OrderItemCreate] = Field(min_length=1, description="Al menos un item")


class OrderResponse(BaseModel):
    """Respuesta completa de un pedido."""
    id: UUID = Field(default_factory=uuid4)
    customer_id: UUID
    status: OrderStatus = OrderStatus.PENDING
    items: list[OrderItemResponse]
    total: float
    created_at: datetime = Field(default_factory=datetime.utcnow)

    model_config = {"json_schema_extra": {
        "examples": [{
            "id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
            "customer_id": "11111111-1111-1111-1111-111111111111",
            "status": "pending",
            "items": [{"product_id": "22222222-2222-2222-2222-222222222222",
                       "quantity": 2, "unit_price": 29.99, "subtotal": 59.98}],
            "total": 59.98,
            "created_at": "2024-06-01T10:00:00"
        }]
    }}
EOF
```

2. Crea los schemas del **InventoryService**:

```bash
cat > ~/microservices-lab/inventory-service/app/schemas/product.py << 'EOF'
"""Schemas del bounded context de Inventario."""
from enum import Enum
from uuid import UUID, uuid4

from pydantic import BaseModel, Field


class Product(BaseModel):
    """Producto dentro del contexto de Inventario (stock y ubicación)."""
    id: UUID = Field(default_factory=uuid4)
    name: str = Field(min_length=1, max_length=200)
    sku: str = Field(min_length=1, max_length=50, pattern=r"^[A-Z0-9\-]+$")
    stock_quantity: int = Field(ge=0)
    reorder_threshold: int = Field(ge=0, default=10)


class ProductCreate(BaseModel):
    """Payload para registrar un nuevo producto en inventario."""
    name: str = Field(min_length=1, max_length=200)
    sku: str = Field(min_length=1, max_length=50, pattern=r"^[A-Z0-9\-]+$")
    stock_quantity: int = Field(ge=0, default=0)
    reorder_threshold: int = Field(ge=0, default=10)


class ReservationStatus(str, Enum):
    PENDING = "pending"
    CONFIRMED = "confirmed"
    RELEASED = "released"


class StockReservationCreate(BaseModel):
    """Solicitud de reserva de stock para un pedido."""
    product_id: UUID
    order_id: UUID
    quantity: int = Field(gt=0)


class StockReservationResponse(BaseModel):
    """Respuesta de una reserva de stock."""
    id: UUID = Field(default_factory=uuid4)
    product_id: UUID
    order_id: UUID
    quantity: int
    status: ReservationStatus = ReservationStatus.PENDING
EOF
```

### Resultado esperado

Cada contexto tiene sus propios schemas Pydantic independientes. El OrderService no importa ni conoce los modelos internos de Inventario y viceversa — solo comparten IDs (UUID) como referencias externas.

### Verificación

```bash
cd ~/microservices-lab/order-service
source .venv/bin/activate
python -c "
from app.schemas.order import OrderCreate, OrderResponse, OrderStatus
print('✅ OrderService schemas válidos')
print(f'   Estados: {[s.value for s in OrderStatus]}')
"
deactivate

cd ~/microservices-lab/inventory-service
source .venv/bin/activate
python -c "
from app.schemas.product import Product, ProductCreate, StockReservationCreate
print('✅ InventoryService schemas válidos')
print(f'   Product fields: {list(Product.model_fields.keys())}')
"
deactivate
```

---

## Paso 4: Implementación de la capa de servicios (lógica de negocio)

**Objetivo:** Crear la lógica de negocio en la capa de servicios, separada de los routers HTTP, usando almacenamiento en memoria como placeholder hasta la integración con PostgreSQL (Lab 02+).

### Instrucciones

1. Implementa el servicio de pedidos:

```bash
cat > ~/microservices-lab/order-service/app/services/order_service.py << 'EOF'
"""Lógica de negocio del contexto de Pedidos."""
from uuid import UUID, uuid4
from datetime import datetime

from app.schemas.order import (
    OrderCreate, OrderResponse, OrderItemResponse, OrderStatus
)


class OrderService:
    """Servicio de dominio para gestión de pedidos (almacenamiento en memoria)."""

    def __init__(self):
        self._orders: dict[UUID, OrderResponse] = {}

    def create_order(self, payload: OrderCreate) -> OrderResponse:
        """Crea un nuevo pedido en estado PENDING."""
        items = [
            OrderItemResponse(
                product_id=item.product_id,
                quantity=item.quantity,
                unit_price=item.unit_price,
                subtotal=round(item.quantity * item.unit_price, 2),
            )
            for item in payload.items
        ]
        total = round(sum(item.subtotal for item in items), 2)

        order = OrderResponse(
            id=uuid4(),
            customer_id=payload.customer_id,
            status=OrderStatus.PENDING,
            items=items,
            total=total,
            created_at=datetime.utcnow(),
        )
        self._orders[order.id] = order
        return order

    def get_order(self, order_id: UUID) -> OrderResponse | None:
        """Obtiene un pedido por ID."""
        return self._orders.get(order_id)

    def list_orders(self) -> list[OrderResponse]:
        """Lista todos los pedidos."""
        return list(self._orders.values())

    def cancel_order(self, order_id: UUID) -> OrderResponse | None:
        """Cancela un pedido si está en estado PENDING."""
        order = self._orders.get(order_id)
        if order is None:
            return None
        if order.status != OrderStatus.PENDING:
            return None
        cancelled = order.model_copy(update={"status": OrderStatus.CANCELLED})
        self._orders[order_id] = cancelled
        return cancelled


# Instancia singleton para el ciclo de vida de la aplicación
order_service = OrderService()
EOF
```

2. Implementa el servicio de inventario:

```bash
cat > ~/microservices-lab/inventory-service/app/services/inventory_service.py << 'EOF'
"""Lógica de negocio del contexto de Inventario."""
from uuid import UUID, uuid4

from app.schemas.product import (
    Product, ProductCreate,
    StockReservationCreate, StockReservationResponse, ReservationStatus
)


class InventoryService:
    """Servicio de dominio para gestión de inventario (almacenamiento en memoria)."""

    def __init__(self):
        self._products: dict[UUID, Product] = {}
        self._reservations: dict[UUID, StockReservationResponse] = {}

    def add_product(self, payload: ProductCreate) -> Product:
        """Registra un nuevo producto en inventario."""
        product = Product(
            id=uuid4(),
            name=payload.name,
            sku=payload.sku,
            stock_quantity=payload.stock_quantity,
            reorder_threshold=payload.reorder_threshold,
        )
        self._products[product.id] = product
        return product

    def get_product(self, product_id: UUID) -> Product | None:
        """Obtiene un producto por ID."""
        return self._products.get(product_id)

    def list_products(self) -> list[Product]:
        """Lista todos los productos."""
        return list(self._products.values())

    def create_reservation(self, payload: StockReservationCreate) -> StockReservationResponse | None:
        """Crea una reserva de stock si hay disponibilidad."""
        product = self._products.get(payload.product_id)
        if product is None:
            return None
        if product.stock_quantity < payload.quantity:
            return None

        # Decrementar stock
        updated = product.model_copy(
            update={"stock_quantity": product.stock_quantity - payload.quantity}
        )
        self._products[product.id] = updated

        reservation = StockReservationResponse(
            id=uuid4(),
            product_id=payload.product_id,
            order_id=payload.order_id,
            quantity=payload.quantity,
            status=ReservationStatus.PENDING,
        )
        self._reservations[reservation.id] = reservation
        return reservation


# Instancia singleton
inventory_service = InventoryService()
EOF
```

### Resultado esperado

Cada servicio encapsula su lógica de negocio en una clase dedicada, independiente del framework HTTP. El almacenamiento en memoria permite prototipar sin dependencias externas.

### Verificación

```bash
cd ~/microservices-lab/order-service && source .venv/bin/activate
python -c "
from uuid import uuid4
from app.schemas.order import OrderCreate, OrderItemCreate
from app.services.order_service import order_service

order = order_service.create_order(OrderCreate(
    customer_id=uuid4(),
    items=[OrderItemCreate(product_id=uuid4(), quantity=3, unit_price=10.0)]
))
print(f'✅ Pedido creado: {order.id}, total: {order.total}, status: {order.status.value}')
"
deactivate
```

Salida esperada:
```
✅ Pedido creado: <uuid>, total: 30.0, status: pending
```

---

## Paso 5: Implementación de routers FastAPI

**Objetivo:** Crear los endpoints RESTful que exponen la funcionalidad de cada servicio, siguiendo el contrato OpenAPI implícito definido por los schemas Pydantic.

### Instrucciones

1. Implementa el router de pedidos:

```bash
cat > ~/microservices-lab/order-service/app/routers/orders.py << 'EOF'
"""Router HTTP para el contexto de Pedidos."""
from uuid import UUID

from fastapi import APIRouter, HTTPException, status

from app.schemas.order import OrderCreate, OrderResponse
from app.services.order_service import order_service

router = APIRouter(prefix="/api/v1/orders", tags=["orders"])


@router.post(
    "",
    response_model=OrderResponse,
    status_code=status.HTTP_201_CREATED,
    summary="Crear un nuevo pedido",
)
def create_order(payload: OrderCreate) -> OrderResponse:
    """Crea un pedido en estado PENDING con los items proporcionados."""
    return order_service.create_order(payload)


@router.get(
    "",
    response_model=list[OrderResponse],
    summary="Listar todos los pedidos",
)
def list_orders() -> list[OrderResponse]:
    """Retorna la lista completa de pedidos."""
    return order_service.list_orders()


@router.get(
    "/{order_id}",
    response_model=OrderResponse,
    summary="Obtener un pedido por ID",
)
def get_order(order_id: UUID) -> OrderResponse:
    """Retorna un pedido específico o 404 si no existe."""
    order = order_service.get_order(order_id)
    if order is None:
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND,
            detail=f"Order {order_id} not found",
        )
    return order


@router.patch(
    "/{order_id}/cancel",
    response_model=OrderResponse,
    summary="Cancelar un pedido",
)
def cancel_order(order_id: UUID) -> OrderResponse:
    """Cancela un pedido en estado PENDING. Retorna 404 o 409 si no es posible."""
    order = order_service.get_order(order_id)
    if order is None:
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND,
            detail=f"Order {order_id} not found",
        )
    cancelled = order_service.cancel_order(order_id)
    if cancelled is None:
        raise HTTPException(
            status_code=status.HTTP_409_CONFLICT,
            detail=f"Order {order_id} cannot be cancelled (status: {order.status.value})",
        )
    return cancelled
EOF
```

2. Implementa el router de inventario:

```bash
cat > ~/microservices-lab/inventory-service/app/routers/products.py << 'EOF'
"""Router HTTP para el contexto de Inventario."""
from uuid import UUID

from fastapi import APIRouter, HTTPException, status

from app.schemas.product import (
    Product, ProductCreate,
    StockReservationCreate, StockReservationResponse,
)
from app.services.inventory_service import inventory_service

router = APIRouter(prefix="/api/v1", tags=["inventory"])


@router.post(
    "/products",
    response_model=Product,
    status_code=status.HTTP_201_CREATED,
    summary="Registrar un producto en inventario",
)
def create_product(payload: ProductCreate) -> Product:
    """Registra un nuevo producto con stock inicial."""
    return inventory_service.add_product(payload)


@router.get(
    "/products",
    response_model=list[Product],
    summary="Listar productos en inventario",
)
def list_products() -> list[Product]:
    """Retorna todos los productos registrados."""
    return inventory_service.list_products()


@router.get(
    "/products/{product_id}",
    response_model=Product,
    summary="Obtener un producto por ID",
)
def get_product(product_id: UUID) -> Product:
    """Retorna un producto específico o 404."""
    product = inventory_service.get_product(product_id)
    if product is None:
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND,
            detail=f"Product {product_id} not found",
        )
    return product


@router.post(
    "/reservations",
    response_model=StockReservationResponse,
    status_code=status.HTTP_201_CREATED,
    summary="Crear una reserva de stock",
)
def create_reservation(payload: StockReservationCreate) -> StockReservationResponse:
    """Reserva stock para un pedido. Retorna 409 si no hay disponibilidad."""
    reservation = inventory_service.create_reservation(payload)
    if reservation is None:
        raise HTTPException(
            status_code=status.HTTP_409_CONFLICT,
            detail="Insufficient stock or product not found",
        )
    return reservation
EOF
```

3. Crea la aplicación principal de **OrderService**:

```bash
cat > ~/microservices-lab/order-service/app/main.py << 'EOF'
"""Punto de entrada de OrderService."""
from fastapi import FastAPI

from app.routers.orders import router as orders_router

app = FastAPI(
    title="OrderService",
    description="Microservicio de gestión de pedidos — Bounded Context: Pedidos",
    version="1.0.0",
    docs_url="/docs",
    openapi_url="/openapi.json",
)

app.include_router(orders_router)


@app.get("/health", tags=["health"])
def health_check():
    """Endpoint de health check."""
    return {"status": "healthy", "service": "order-service"}
EOF
```

4. Crea la aplicación principal de **InventoryService**:

```bash
cat > ~/microservices-lab/inventory-service/app/main.py << 'EOF'
"""Punto de entrada de InventoryService."""
from fastapi import FastAPI

from app.routers.products import router as products_router

app = FastAPI(
    title="InventoryService",
    description="Microservicio de inventario — Bounded Context: Inventario",
    version="1.0.0",
    docs_url="/docs",
    openapi_url="/openapi.json",
)

app.include_router(products_router)


@app.get("/health", tags=["health"])
def health_check():
    """Endpoint de health check."""
    return {"status": "healthy", "service": "inventory-service"}
EOF
```

### Resultado esperado

Ambos servicios tienen aplicaciones FastAPI funcionales con routers, health checks y documentación OpenAPI automática.

### Verificación

Inicia temporalmente el OrderService para validar:

```bash
cd ~/microservices-lab/order-service && source .venv/bin/activate
uvicorn app.main:app --host 0.0.0.0 --port 8001 &
SERVER_PID=$!
sleep 2

# Verificar health
curl -s http://localhost:8001/health | python -m json.tool

# Verificar OpenAPI schema
curl -s http://localhost:8001/openapi.json | python -c "
import json, sys
schema = json.load(sys.stdin)
paths = list(schema['paths'].keys())
print(f'✅ OrderService OpenAPI generado con {len(paths)} paths: {paths}')
"

kill $SERVER_PID 2>/dev/null
deactivate
```

Salida esperada:
```json
{
    "status": "healthy",
    "service": "order-service"
}
```
```
✅ OrderService OpenAPI generado con 4 paths: ['/api/v1/orders', '/api/v1/orders/{order_id}', '/api/v1/orders/{order_id}/cancel', '/health']
```

---

## Paso 6: Pruebas contractuales con schemathesis

**Objetivo:** Validar que la implementación de cada servicio es conforme a su propio contrato OpenAPI mediante pruebas contractuales automatizadas.

### Instrucciones

1. Crea el archivo de pruebas contractuales para **OrderService**:

```bash
cat > ~/microservices-lab/order-service/tests/test_contract.py << 'EOF'
"""Pruebas contractuales: valida que la implementación cumple el contrato OpenAPI."""
import schemathesis
from fastapi.testclient import TestClient

from app.main import app

# Genera los test cases a partir del schema OpenAPI de la propia aplicación
schema = schemathesis.from_asgi("/openapi.json", app=app)


@schema.parametrize()
def test_api_contract(case):
    """
    Schemathesis genera automáticamente peticiones válidas e inválidas
    basándose en el schema OpenAPI y verifica que:
    - Las respuestas exitosas cumplen el schema de respuesta declarado
    - No se producen errores 500 inesperados
    - Los códigos de estado son coherentes con la especificación
    """
    response = case.as_transport_kwargs()
    with TestClient(app) as client:
        result = case.as_transport_kwargs()
        resp = client.request(
            method=case.method.upper(),
            url=case.formatted_path,
            headers=case.headers,
            json=case.body,
            params=case.query,
        )
    case.validate_response(resp, additional_checks=())
EOF
```

2. Crea un test funcional adicional con httpx para **OrderService**:

```bash
cat > ~/microservices-lab/order-service/tests/test_orders_functional.py << 'EOF'
"""Tests funcionales del endpoint de pedidos."""
import pytest
from httpx import AsyncClient, ASGITransport

from app.main import app


@pytest.fixture
def anyio_backend():
    return "asyncio"


@pytest.mark.asyncio
async def test_create_order_success():
    """Verifica la creación exitosa de un pedido."""
    transport = ASGITransport(app=app)
    async with AsyncClient(transport=transport, base_url="http://test") as client:
        payload = {
            "customer_id": "11111111-1111-1111-1111-111111111111",
            "items": [
                {
                    "product_id": "22222222-2222-2222-2222-222222222222",
                    "quantity": 2,
                    "unit_price": 25.50,
                }
            ],
        }
        response = await client.post("/api/v1/orders", json=payload)

    assert response.status_code == 201
    data = response.json()
    assert data["status"] == "pending"
    assert data["total"] == 51.0
    assert len(data["items"]) == 1
    assert data["items"][0]["subtotal"] == 51.0


@pytest.mark.asyncio
async def test_create_order_empty_items_fails():
    """Verifica que no se puede crear un pedido sin items."""
    transport = ASGITransport(app=app)
    async with AsyncClient(transport=transport, base_url="http://test") as client:
        payload = {
            "customer_id": "11111111-1111-1111-1111-111111111111",
            "items": [],
        }
        response = await client.post("/api/v1/orders", json=payload)

    assert response.status_code == 422  # Validation error


@pytest.mark.asyncio
async def test_get_nonexistent_order_returns_404():
    """Verifica que consultar un pedido inexistente retorna 404."""
    transport = ASGITransport(app=app)
    async with AsyncClient(transport=transport, base_url="http://test") as client:
        response = await client.get(
            "/api/v1/orders/99999999-9999-9999-9999-999999999999"
        )

    assert response.status_code == 404
EOF
```

3. Crea las pruebas contractuales para **InventoryService**:

```bash
cat > ~/microservices-lab/inventory-service/tests/test_contract.py << 'EOF'
"""Pruebas contractuales para InventoryService."""
import schemathesis
from fastapi.testclient import TestClient

from app.main import app

schema = schemathesis.from_asgi("/openapi.json", app=app)


@schema.parametrize()
def test_api_contract(case):
    """Valida conformidad de la implementación con el contrato OpenAPI."""
    with TestClient(app) as client:
        resp = client.request(
            method=case.method.upper(),
            url=case.formatted_path,
            headers=case.headers,
            json=case.body,
            params=case.query,
        )
    case.validate_response(resp, additional_checks=())
EOF
```

```bash
cat > ~/microservices-lab/inventory-service/tests/test_inventory_functional.py << 'EOF'
"""Tests funcionales del endpoint de inventario."""
import pytest
from httpx import AsyncClient, ASGITransport

from app.main import app


@pytest.mark.asyncio
async def test_create_product_success():
    """Verifica registro exitoso de un producto."""
    transport = ASGITransport(app=app)
    async with AsyncClient(transport=transport, base_url="http://test") as client:
        payload = {
            "name": "Widget Premium",
            "sku": "WDG-001",
            "stock_quantity": 100,
            "reorder_threshold": 20,
        }
        response = await client.post("/api/v1/products", json=payload)

    assert response.status_code == 201
    data = response.json()
    assert data["name"] == "Widget Premium"
    assert data["stock_quantity"] == 100


@pytest.mark.asyncio
async def test_create_product_invalid_sku_fails():
    """Verifica que un SKU con formato inválido es rechazado."""
    transport = ASGITransport(app=app)
    async with AsyncClient(transport=transport, base_url="http://test") as client:
        payload = {
            "name": "Bad Product",
            "sku": "invalid sku!",  # No cumple pattern ^[A-Z0-9\-]+$
            "stock_quantity": 10,
        }
        response = await client.post("/api/v1/products", json=payload)

    assert response.status_code == 422


@pytest.mark.asyncio
async def test_reservation_insufficient_stock():
    """Verifica que una reserva sin stock suficiente retorna 409."""
    transport = ASGITransport(app=app)
    async with AsyncClient(transport=transport, base_url="http://test") as client:
        # Crear producto con stock limitado
        product_resp = await client.post("/api/v1/products", json={
            "name": "Limited Item", "sku": "LTD-001", "stock_quantity": 5,
        })
        product_id = product_resp.json()["id"]

        # Intentar reservar más del disponible
        reservation_resp = await client.post("/api/v1/reservations", json={
            "product_id": product_id,
            "order_id": "33333333-3333-3333-3333-333333333333",
            "quantity": 999,
        })

    assert reservation_resp.status_code == 409
EOF
```

4. Ejecuta las pruebas de **OrderService**:

```bash
cd ~/microservices-lab/order-service && source .venv/bin/activate
python -m pytest tests/ -v --tb=short
deactivate
```

5. Ejecuta las pruebas de **InventoryService**:

```bash
cd ~/microservices-lab/inventory-service && source .venv/bin/activate
python -m pytest tests/ -v --tb=short
deactivate
```

### Resultado esperado

Todas las pruebas pasan exitosamente. Las pruebas contractuales de schemathesis generan múltiples casos automáticamente y verifican que ninguna respuesta viola el schema OpenAPI declarado.

### Verificación

La salida de pytest debe mostrar:
```
========== X passed in Y.YYs ==========
```

Sin fallos ni errores. Los tests contractuales (`test_api_contract`) aparecerán con múltiples parametrizaciones automáticas.

---

## Paso 7: Dockerización de los servicios

**Objetivo:** Crear Dockerfiles optimizados y un docker-compose.yml que permita ejecutar ambos servicios simultáneamente.

### Instrucciones

1. Crea el Dockerfile para **OrderService**:

```bash
cat > ~/microservices-lab/order-service/Dockerfile << 'EOF'
FROM python:3.12.3-slim AS base

# Evitar bytecode y buffering
ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1

WORKDIR /app

# Instalar dependencias primero (capa cacheada)
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copiar código fuente
COPY app/ ./app/

# Usuario no-root
RUN adduser --disabled-password --no-create-home appuser
USER appuser

EXPOSE 8001

CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8001"]
EOF
```

2. Crea el Dockerfile para **InventoryService**:

```bash
cat > ~/microservices-lab/inventory-service/Dockerfile << 'EOF'
FROM python:3.12.3-slim AS base

ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY app/ ./app/

RUN adduser --disabled-password --no-create-home appuser
USER appuser

EXPOSE 8002

CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8002"]
EOF
```

3. Crea el archivo `.dockerignore` para ambos servicios:

```bash
for svc in order-service inventory-service; do
cat > ~/microservices-lab/$svc/.dockerignore << 'EOF'
.venv/
__pycache__/
*.pyc
.pytest_cache/
tests/
.git/
EOF
done
```

4. Crea el `docker-compose.yml` en la raíz del proyecto:

```bash
cat > ~/microservices-lab/docker-compose.yml << 'EOF'
version: "3.9"

services:
  mslab-order-service:
    container_name: mslab-order-service
    build:
      context: ./order-service
      dockerfile: Dockerfile
    ports:
      - "8001:8001"
    healthcheck:
      test: ["CMD", "python", "-c", "import urllib.request; urllib.request.urlopen('http://localhost:8001/health')"]
      interval: 10s
      timeout: 5s
      retries: 3
    networks:
      - mslab-network

  mslab-inventory-service:
    container_name: mslab-inventory-service
    build:
      context: ./inventory-service
      dockerfile: Dockerfile
    ports:
      - "8002:8002"
    healthcheck:
      test: ["CMD", "python", "-c", "import urllib.request; urllib.request.urlopen('http://localhost:8002/health')"]
      interval: 10s
      timeout: 5s
      retries: 3
    networks:
      - mslab-network

networks:
  mslab-network:
    driver: bridge
EOF
```

5. Construye y levanta los servicios:

```bash
cd ~/microservices-lab
docker compose up --build -d
```

6. Espera a que los servicios estén healthy:

```bash
echo "Esperando a que los servicios estén listos..."
sleep 10
docker compose ps
```

### Resultado esperado

```
NAME                      IMAGE                              STATUS                   PORTS
mslab-inventory-service   microservices-lab-mslab-inventory   Up X seconds (healthy)   0.0.0.0:8002->8002/tcp
mslab-order-service       microservices-lab-mslab-order       Up X seconds (healthy)   0.0.0.0:8001->8001/tcp
```

### Verificación

```bash
# Health checks
echo "--- OrderService ---"
curl -s http://localhost:8001/health | python -m json.tool

echo "--- InventoryService ---"
curl -s http://localhost:8002/health | python -m json.tool

# Test funcional end-to-end: crear un pedido
echo "--- Crear pedido ---"
curl -s -X POST http://localhost:8001/api/v1/orders \
  -H "Content-Type: application/json" \
  -d '{
    "customer_id": "11111111-1111-1111-1111-111111111111",
    "items": [{"product_id": "22222222-2222-2222-2222-222222222222", "quantity": 3, "unit_price": 19.99}]
  }' | python -m json.tool

# Test funcional: crear un producto en inventario
echo "--- Crear producto ---"
curl -s -X POST http://localhost:8002/api/v1/products \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Sensor IoT v2",
    "sku": "SNS-IOT-002",
    "stock_quantity": 250,
    "reorder_threshold": 50
  }' | python -m json.tool
```

Salida esperada del pedido:
```json
{
    "id": "<uuid>",
    "customer_id": "11111111-1111-1111-1111-111111111111",
    "status": "pending",
    "items": [
        {
            "product_id": "22222222-2222-2222-2222-222222222222",
            "quantity": 3,
            "unit_price": 19.99,
            "subtotal": 59.97
        }
    ],
    "total": 59.97,
    "created_at": "<timestamp>"
}
```

---

## Paso 8: Exportación y validación del contrato OpenAPI

**Objetivo:** Exportar los esquemas OpenAPI generados a archivos estáticos para documentación y como contratos formales versionables en el repositorio.

### Instrucciones

1. Exporta los schemas OpenAPI de ambos servicios:

```bash
mkdir -p ~/microservices-lab/docs/api

# Exportar OpenAPI de OrderService
curl -s http://localhost:8001/openapi.json | python -m json.tool > ~/microservices-lab/docs/api/order-service-openapi.json

# Exportar OpenAPI de InventoryService
curl -s http://localhost:8002/openapi.json | python -m json.tool > ~/microservices-lab/docs/api/inventory-service-openapi.json

echo "✅ Schemas exportados"
ls -la ~/microservices-lab/docs/api/
```

2. Valida la estructura de los schemas exportados:

```bash
python -c "
import json

for svc in ['order-service', 'inventory-service']:
    with open(f'$HOME/microservices-lab/docs/api/{svc}-openapi.json') as f:
        schema = json.load(f)
    
    print(f'\\n=== {svc} ===')
    print(f'  Title: {schema[\"info\"][\"title\"]}')
    print(f'  Version: {schema[\"info\"][\"version\"]}')
    print(f'  Paths: {list(schema[\"paths\"].keys())}')
    print(f'  Schemas: {list(schema[\"components\"][\"schemas\"].keys())}')
"
```

### Resultado esperado

```
=== order-service ===
  Title: OrderService
  Version: 1.0.0
  Paths: ['/api/v1/orders', '/api/v1/orders/{order_id}', '/api/v1/orders/{order_id}/cancel', '/health']
  Schemas: ['OrderCreate', 'OrderItemCreate', 'OrderItemResponse', 'OrderResponse', 'OrderStatus', ...]

=== inventory-service ===
  Title: InventoryService
  Version: 1.0.0
  Paths: ['/api/v1/products', '/api/v1/products/{product_id}', '/api/v1/reservations', '/health']
  Schemas: ['Product', 'ProductCreate', 'StockReservationCreate', 'StockReservationResponse', ...]
```

### Verificación

```bash
test -f ~/microservices-lab/docs/api/order-service-openapi.json && \
test -f ~/microservices-lab/docs/api/inventory-service-openapi.json && \
echo "✅ Ambos contratos OpenAPI exportados y disponibles para versionamiento"
```

---

## Validación y pruebas finales

Ejecuta esta secuencia completa para verificar que todo el laboratorio está correcto:

```bash
cd ~/microservices-lab

echo "╔══════════════════════════════════════════════════════╗"
echo "║  VALIDACIÓN FINAL — Lab 01-00-01                    ║"
echo "╚══════════════════════════════════════════════════════╝"

# 1. Verificar estructura de directorios
echo ""
echo "1️⃣ Estructura del proyecto:"
for dir in order-service/app/{routers,services,schemas,models} \
           inventory-service/app/{routers,services,schemas,models} \
           docs/api tests; do
    test -d "$dir" && echo "   ✅ $dir" || echo "   ❌ $dir FALTANTE"
done

# 2. Verificar contenedores running
echo ""
echo "2️⃣ Contenedores Docker:"
docker compose ps --format "table {{.Name}}\t{{.Status}}\t{{.Ports}}"

# 3. Health checks
echo ""
echo "3️⃣ Health checks:"
ORDER_HEALTH=$(curl -s -o /dev/null -w "%{http_code}" http://localhost:8001/health)
INVENTORY_HEALTH=$(curl -s -o /dev/null -w "%{http_code}" http://localhost:8002/health)
echo "   OrderService:     HTTP $ORDER_HEALTH"
echo "   InventoryService: HTTP $INVENTORY_HEALTH"

# 4. Pruebas unitarias
echo ""
echo "4️⃣ Pruebas OrderService:"
cd ~/microservices-lab/order-service && source .venv/bin/activate
python -m pytest tests/test_orders_functional.py -v --tb=short 2>&1 | tail -5
deactivate

echo ""
echo "5️⃣ Pruebas InventoryService:"
cd ~/microservices-lab/inventory-service && source .venv/bin/activate
python -m pytest tests/test_inventory_functional.py -v --tb=short 2>&1 | tail -5
deactivate

# 5. Contratos OpenAPI
echo ""
echo "6️⃣ Contratos OpenAPI:"
test -f ~/microservices-lab/docs/api/order-service-openapi.json && echo "   ✅ order-service-openapi.json"
test -f ~/microservices-lab/docs/api/inventory-service-openapi.json && echo "   ✅ inventory-service-openapi.json"

echo ""
echo "══════════════════════════════════════════════════════"
echo "  ✅ LABORATORIO COMPLETADO EXITOSAMENTE"
echo "══════════════════════════════════════════════════════"
```

---

## Resolución de Problemas

### Problema 1: Error `ModuleNotFoundError: No module named 'app'` al ejecutar pytest

**Síntomas:** Al ejecutar `python -m pytest tests/` se obtiene un error de importación indicando que el módulo `app` no se encuentra.

**Causa:** pytest no encuentra el paquete `app` porque el directorio de trabajo no está en el `sys.path`. Esto ocurre cuando se ejecuta pytest desde un directorio incorrecto o falta configuración de proyecto.

**Solución:**

```bash
# Opción 1: Asegurarse de ejecutar desde la raíz del servicio
cd ~/microservices-lab/order-service
source .venv/bin/activate
python -m pytest tests/ -v

# Opción 2: Crear un pyproject.toml mínimo que configure el path
cat > ~/microservices-lab/order-service/pyproject.toml << 'EOF'
[tool.pytest.ini_options]
pythonpath = ["."]
testpaths = ["tests"]
asyncio_mode = "auto"
EOF
```

Repite para `inventory-service` si es necesario.

---

### Problema 2: Puerto 8001 o 8002 ya en uso al levantar docker compose

**Síntomas:** `docker compose up` falla con error `Bind for 0.0.0.0:8001 failed: port is already allocated`.

**Causa:** Un proceso previo de uvicorn (del paso de verificación manual) o otro contenedor sigue ocupando el puerto.

**Solución:**

```bash
# Identificar qué proceso usa el puerto
lsof -i :8001
# o
ss -tlnp | grep 8001

# Si es un proceso uvicorn local, matarlo
pkill -f "uvicorn app.main:app.*8001"

# Si es un contenedor Docker previo
docker ps -a | grep 8001
docker stop <container_id> && docker rm <container_id>

# Reintentar
cd ~/microservices-lab
docker compose up --build -d
```

---

## Limpieza

Para detener y eliminar los recursos creados durante el laboratorio:

```bash
cd ~/microservices-lab

# Detener y eliminar contenedores
docker compose down --rmi local --volumes

# (OPCIONAL) Si deseas eliminar todo el workspace para empezar de nuevo:
# rm -rf ~/microservices-lab

# Verificar limpieza
docker ps -a | grep mslab || echo "✅ No quedan contenedores del lab"
```

> **⚠️ Nota:** NO elimines `~/microservices-lab/` si vas a continuar con el Lab 02. Este directorio es el workspace permanente del curso.

---

## Resumen

### Lo que construiste

| Componente | Descripción |
|------------|-------------|
| **Bounded Contexts** | Documentación formal de los dominios Pedidos e Inventario con fronteras claras |
| **OrderService** | Microservicio FastAPI (puerto 8001) con CRUD de pedidos y lógica de cancelación |
| **InventoryService** | Microservicio FastAPI (puerto 8002) con gestión de productos y reservas de stock |
| **Schemas Pydantic v2** | Modelos de dominio tipados con validación estricta por contexto |
| **Pruebas contractuales** | Validación automatizada con schemathesis contra el schema OpenAPI |
| **Dockerización** | Dockerfiles optimizados y docker-compose.yml para ejecución simultánea |
| **Contratos OpenAPI** | Schemas exportados como artefactos versionables |

### Principios aplicados

- **API-first:** Los schemas Pydantic definen el contrato antes de la implementación
- **Bounded Context:** Cada servicio tiene sus propios modelos — sin compartir entidades
- **Separación de capas:** routers → services → schemas (sin acoplamiento directo)
- **Autonomía de despliegue:** Cada servicio tiene su propio Dockerfile y ciclo de vida

### Próximos pasos

En el **Lab 02** integrarás Apache Kafka para comunicación asíncrona entre los servicios, implementando el patrón de eventos de dominio que permite a OrderService notificar a InventoryService sobre nuevos pedidos sin acoplamiento directo.

### Recursos adicionales

- [FastAPI Documentation — OpenAPI](https://fastapi.tiangolo.com/tutorial/metadata/)
- [Pydantic v2 — Model Validators](https://docs.pydantic.dev/latest/concepts/validators/)
- [Schemathesis — Stateful Testing](https://schemathesis.readthedocs.io/en/stable/)
- [Eric Evans — Domain-Driven Design Reference](https://www.domainlanguage.com/ddd/reference/)
