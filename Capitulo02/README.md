# Integrar Microservicios Python con Kafka y Garantizar Idempotencia

## Metadata

| Campo | Valor |
|-------|-------|
| **Duración** | 57 minutos |
| **Complejidad** | Alta |
| **Nivel Bloom** | Aplicar |

## Descripción General

En este laboratorio implementarás una capa de mensajería asíncrona entre OrderService e InventoryService usando Apache Kafka como broker de eventos. Diseñarás esquemas de eventos de dominio (`OrderCreated`, `StockReserved`, `StockReservationFailed`) con identificadores únicos UUID v4 y versionado semántico. Implementarás idempotencia en los consumidores mediante Redis como registro de deduplicación con TTL de 24 horas, y validarás que el sistema maneja correctamente la reentrega de mensajes sin duplicar efectos laterales.

## Objetivos de Aprendizaje

- [ ] Integrar OrderService e InventoryService mediante Apache Kafka usando productores y consumidores asíncronos con `confluent-kafka-python`
- [ ] Diseñar esquemas de eventos de dominio con identificadores únicos (UUID v4), timestamps y versionado semántico usando Pydantic v2
- [ ] Implementar idempotencia en consumidores usando Redis como registro de eventos procesados con TTL de 24 horas
- [ ] Configurar topics Kafka con particionamiento por `order_id` para garantizar orden de eventos por entidad
- [ ] Validar el comportamiento del sistema bajo reentrega de mensajes duplicados

## Prerrequisitos

### Conocimiento Requerido

- Lab 01 completado: OrderService (puerto 8001) e InventoryService (puerto 8002) operativos
- Comprensión de conceptos de mensajería: topics, particiones, consumer groups, offsets
- Conocimiento de `async/await` en Python y FastAPI
- Familiaridad con Docker Compose y gestión de contenedores

### Acceso Requerido

- Directorio `~/microservices-lab/` con estructura del Lab 01
- Docker Engine 26.1.3+ y Docker Compose 2.27.1+
- Acceso a internet para descargar imágenes de contenedores

## Entorno del Laboratorio

### Software Adicional Requerido

| Componente | Versión | Imagen/Paquete |
|------------|---------|----------------|
| Apache Kafka | 7.6.1 | `confluentinc/cp-kafka:7.6.1` |
| Apache Zookeeper | 7.6.1 | `confluentinc/cp-zookeeper:7.6.1` |
| Redis | 7.2.5 | `redis:7.2.5-alpine` |
| confluent-kafka-python | 2.4.0 | `pip: confluent-kafka==2.4.0` |
| redis-py (async) | 5.0.4 | `pip: redis==5.0.4` |
| pytest | 8.2.1 | `pip: pytest==8.2.1` |
| pytest-asyncio | 0.23.7 | `pip: pytest-asyncio==0.23.7` |

### Arquitectura del Laboratorio

```
┌─────────────────┐    orders.created     ┌─────────────────────┐
│  OrderService   │ ──────────────────►   │  InventoryService   │
│  (puerto 8001)  │                       │  (puerto 8002)      │
└─────────────────┘                       └──────────┬──────────┘
                                                     │
                              ┌───────────────────────┼───────────────────────┐
                              │                       │                       │
                              ▼                       ▼                       ▼
                 inventory.stock-reserved  inventory.stock-failed        Redis
                                                                    (deduplicación)
```

## Paso 1: Extender Docker Compose con Kafka, Zookeeper y Redis

**Objetivo:** Añadir los servicios de infraestructura necesarios (Kafka, Zookeeper, Redis) al entorno existente del Lab 01.

### Instrucciones

1. Navega al directorio base del proyecto:

```bash
cd ~/microservices-lab/
```

2. Crea un archivo `docker-compose.kafka.yml` que extienda la configuración base:

```yaml
# ~/microservices-lab/docker-compose.kafka.yml
version: "3.9"

services:
  mslab-zookeeper:
    image: confluentinc/cp-zookeeper:7.6.1
    container_name: mslab-zookeeper
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
      ZOOKEEPER_TICK_TIME: 2000
    ports:
      - "2181:2181"
    healthcheck:
      test: ["CMD", "echo", "ruok", "|", "nc", "localhost", "2181"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - mslab-network

  mslab-kafka:
    image: confluentinc/cp-kafka:7.6.1
    container_name: mslab-kafka
    depends_on:
      mslab-zookeeper:
        condition: service_healthy
    ports:
      - "9092:9092"
      - "29092:29092"
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: mslab-zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://mslab-kafka:9092,PLAINTEXT_HOST://localhost:29092
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: PLAINTEXT:PLAINTEXT,PLAINTEXT_HOST:PLAINTEXT
      KAFKA_INTER_BROKER_LISTENER_NAME: PLAINTEXT
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
      KAFKA_AUTO_CREATE_TOPICS_ENABLE: "false"
    healthcheck:
      test: ["CMD", "kafka-broker-api-versions", "--bootstrap-server", "localhost:9092"]
      interval: 10s
      timeout: 10s
      retries: 10
    networks:
      - mslab-network

  mslab-redis:
    image: redis:7.2.5-alpine
    container_name: mslab-redis
    ports:
      - "6379:6379"
    command: redis-server --maxmemory 128mb --maxmemory-policy allkeys-lru
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 3s
      retries: 5
    networks:
      - mslab-network

networks:
  mslab-network:
    driver: bridge
```

3. Levanta los servicios de infraestructura:

```bash
docker compose -f docker-compose.kafka.yml up -d
```

4. Espera a que Kafka esté completamente operativo (puede tomar 30-60 segundos):

```bash
docker compose -f docker-compose.kafka.yml ps
```

5. Crea los topics Kafka con 3 particiones cada uno:

```bash
docker exec mslab-kafka kafka-topics --create \
  --bootstrap-server localhost:9092 \
  --topic orders.created \
  --partitions 3 \
  --replication-factor 1

docker exec mslab-kafka kafka-topics --create \
  --bootstrap-server localhost:9092 \
  --topic inventory.stock-reserved \
  --partitions 3 \
  --replication-factor 1

docker exec mslab-kafka kafka-topics --create \
  --bootstrap-server localhost:9092 \
  --topic inventory.stock-failed \
  --partitions 3 \
  --replication-factor 1
```

6. Verifica que los topics se crearon correctamente:

```bash
docker exec mslab-kafka kafka-topics --list --bootstrap-server localhost:9092
```

### Salida Esperada

```
inventory.stock-failed
inventory.stock-reserved
orders.created
```

### Verificación

```bash
# Verificar que los tres servicios están healthy
docker exec mslab-redis redis-cli ping
# Respuesta: PONG

docker exec mslab-kafka kafka-topics --describe \
  --bootstrap-server localhost:9092 \
  --topic orders.created
# Debe mostrar 3 particiones
```

## Paso 2: Definir Esquemas de Eventos de Dominio

**Objetivo:** Crear modelos Pydantic v2 para los eventos de dominio con identificadores únicos UUID v4, timestamps y versionado semántico.

### Instrucciones

1. Crea la estructura de directorios para los módulos compartidos de eventos:

```bash
mkdir -p ~/microservices-lab/shared/events
touch ~/microservices-lab/shared/__init__.py
touch ~/microservices-lab/shared/events/__init__.py
```

2. Crea el módulo base de eventos en `~/microservices-lab/shared/events/base.py`:

```python
# ~/microservices-lab/shared/events/base.py
"""Módulo base para eventos de dominio con soporte de versionado e idempotencia."""

from datetime import datetime, timezone
from typing import Any
from uuid import UUID, uuid4

from pydantic import BaseModel, Field


class DomainEvent(BaseModel):
    """Clase base para todos los eventos de dominio.

    Cada evento incluye:
    - event_id: identificador único UUID v4 para deduplicación
    - event_type: tipo del evento (ej: 'OrderCreated')
    - event_version: versión semántica del esquema del evento
    - timestamp: momento de creación del evento en UTC ISO 8601
    - correlation_id: ID para rastrear la cadena de eventos relacionados
    - source_service: nombre del servicio que originó el evento
    """

    event_id: UUID = Field(default_factory=uuid4, description="ID único del evento para idempotencia")
    event_type: str = Field(..., description="Tipo del evento de dominio")
    event_version: str = Field(default="1.0.0", description="Versión semántica del esquema")
    timestamp: datetime = Field(
        default_factory=lambda: datetime.now(timezone.utc),
        description="Timestamp UTC de creación del evento",
    )
    correlation_id: UUID = Field(default_factory=uuid4, description="ID de correlación para trazabilidad")
    source_service: str = Field(..., description="Servicio que emitió el evento")

    def to_kafka_value(self) -> bytes:
        """Serializa el evento a bytes JSON para Kafka."""
        return self.model_dump_json().encode("utf-8")

    @classmethod
    def from_kafka_value(cls, data: bytes) -> "DomainEvent":
        """Deserializa un evento desde bytes JSON de Kafka."""
        return cls.model_validate_json(data)
```

3. Crea los esquemas específicos de eventos en `~/microservices-lab/shared/events/order_events.py`:

```python
# ~/microservices-lab/shared/events/order_events.py
"""Eventos de dominio relacionados con el servicio de pedidos."""

from uuid import UUID, uuid4

from pydantic import Field

from .base import DomainEvent


class OrderCreated(DomainEvent):
    """Evento emitido cuando se crea un nuevo pedido exitosamente.

    Contiene la información necesaria para que InventoryService
    pueda intentar reservar el stock correspondiente.
    """

    event_type: str = Field(default="OrderCreated", frozen=True)
    source_service: str = Field(default="order-service", frozen=True)

    # Datos del dominio
    order_id: UUID = Field(..., description="ID del pedido creado")
    customer_id: str = Field(..., description="ID del cliente que realizó el pedido")
    product_id: str = Field(..., description="ID del producto solicitado")
    quantity: int = Field(..., gt=0, description="Cantidad solicitada")
```

4. Crea los eventos de inventario en `~/microservices-lab/shared/events/inventory_events.py`:

```python
# ~/microservices-lab/shared/events/inventory_events.py
"""Eventos de dominio relacionados con el servicio de inventario."""

from uuid import UUID

from pydantic import Field

from .base import DomainEvent


class StockReserved(DomainEvent):
    """Evento emitido cuando el stock se reserva exitosamente para un pedido."""

    event_type: str = Field(default="StockReserved", frozen=True)
    source_service: str = Field(default="inventory-service", frozen=True)

    # Datos del dominio
    order_id: UUID = Field(..., description="ID del pedido para el cual se reservó stock")
    product_id: str = Field(..., description="ID del producto reservado")
    quantity_reserved: int = Field(..., gt=0, description="Cantidad reservada")
    remaining_stock: int = Field(..., ge=0, description="Stock restante después de la reserva")


class StockReservationFailed(DomainEvent):
    """Evento emitido cuando no es posible reservar el stock solicitado."""

    event_type: str = Field(default="StockReservationFailed", frozen=True)
    source_service: str = Field(default="inventory-service", frozen=True)

    # Datos del dominio
    order_id: UUID = Field(..., description="ID del pedido que no pudo ser satisfecho")
    product_id: str = Field(..., description="ID del producto solicitado")
    quantity_requested: int = Field(..., gt=0, description="Cantidad solicitada")
    available_stock: int = Field(..., ge=0, description="Stock disponible al momento del intento")
    reason: str = Field(..., description="Razón del fallo de reserva")
```

5. Actualiza el `__init__.py` del paquete de eventos:

```python
# ~/microservices-lab/shared/events/__init__.py
"""Paquete de eventos de dominio compartidos entre microservicios."""

from .base import DomainEvent
from .inventory_events import StockReservationFailed, StockReserved
from .order_events import OrderCreated

__all__ = [
    "DomainEvent",
    "OrderCreated",
    "StockReserved",
    "StockReservationFailed",
]
```

### Salida Esperada

Estructura de archivos creada:

```
~/microservices-lab/shared/
├── __init__.py
└── events/
    ├── __init__.py
    ├── base.py
    ├── inventory_events.py
    └── order_events.py
```

### Verificación

```bash
cd ~/microservices-lab/
python3 -c "
from shared.events import OrderCreated, StockReserved, StockReservationFailed
from uuid import uuid4

event = OrderCreated(
    order_id=uuid4(),
    customer_id='cust-001',
    product_id='prod-abc',
    quantity=5,
    correlation_id=uuid4()
)
print(f'Event type: {event.event_type}')
print(f'Event ID: {event.event_id}')
print(f'Version: {event.event_version}')
print(f'Serialized size: {len(event.to_kafka_value())} bytes')
"
```

Debe imprimir el tipo de evento, un UUID, versión `1.0.0` y el tamaño en bytes.

## Paso 3: Implementar el Productor Kafka en OrderService

**Objetivo:** Crear un productor Kafka asíncrono que publique eventos `OrderCreated` al topic `orders.created` con particionamiento por `order_id`.

### Instrucciones

1. Instala las dependencias necesarias en el entorno de OrderService:

```bash
cd ~/microservices-lab/order-service/
pip install confluent-kafka==2.4.0 redis==5.0.4
```

2. Crea el módulo del productor Kafka en `~/microservices-lab/order-service/kafka_producer.py`:

```python
# ~/microservices-lab/order-service/kafka_producer.py
"""Productor Kafka para OrderService.

Publica eventos de dominio al broker Kafka con particionamiento
por order_id para garantizar orden de eventos por entidad.
"""

import logging
from typing import Optional

from confluent_kafka import KafkaError, Producer

logger = logging.getLogger(__name__)


class OrderEventProducer:
    """Productor de eventos Kafka para el servicio de pedidos.

    Usa confluent-kafka-python con entrega garantizada (acks=all)
    y particionamiento consistente por order_id.
    """

    def __init__(self, bootstrap_servers: str = "localhost:29092"):
        self._config = {
            "bootstrap.servers": bootstrap_servers,
            "client.id": "order-service-producer",
            "acks": "all",  # Esperar confirmación de todas las réplicas
            "retries": 3,
            "retry.backoff.ms": 100,
            "enable.idempotence": True,  # Garantizar exactly-once en el productor
        }
        self._producer: Optional[Producer] = None

    def _get_producer(self) -> Producer:
        """Inicialización lazy del productor."""
        if self._producer is None:
            self._producer = Producer(self._config)
            logger.info("Kafka producer initialized with config: %s", self._config)
        return self._producer

    def _delivery_callback(self, err: Optional[KafkaError], msg) -> None:
        """Callback invocado cuando Kafka confirma o rechaza un mensaje."""
        if err is not None:
            logger.error(
                "Error al entregar mensaje al topic '%s': %s",
                msg.topic(),
                err,
            )
        else:
            logger.info(
                "Evento entregado: topic='%s', partition=%d, offset=%d",
                msg.topic(),
                msg.partition(),
                msg.offset(),
            )

    def publish_order_created(self, event_data: bytes, order_id: str) -> None:
        """Publica un evento OrderCreated al topic 'orders.created'.

        Args:
            event_data: Evento serializado como bytes JSON.
            order_id: ID del pedido, usado como key para particionamiento.
        """
        producer = self._get_producer()
        producer.produce(
            topic="orders.created",
            key=order_id.encode("utf-8"),  # Particionamiento por order_id
            value=event_data,
            callback=self._delivery_callback,
        )
        # Forzar el envío inmediato (flush con timeout)
        producer.flush(timeout=5.0)
        logger.info("Evento OrderCreated publicado para order_id=%s", order_id)

    def close(self) -> None:
        """Cierra el productor y libera recursos."""
        if self._producer is not None:
            remaining = self._producer.flush(timeout=10.0)
            if remaining > 0:
                logger.warning("%d mensajes no pudieron ser entregados al cerrar", remaining)
            self._producer = None
            logger.info("Kafka producer cerrado")


# Instancia singleton del productor
order_producer = OrderEventProducer()
```

3. Crea el endpoint de integración que publica eventos al crear un pedido. Crea `~/microservices-lab/order-service/routes_kafka.py`:

```python
# ~/microservices-lab/order-service/routes_kafka.py
"""Rutas de OrderService con integración Kafka."""

import logging
from uuid import UUID, uuid4

from fastapi import APIRouter, HTTPException, status
from pydantic import BaseModel, Field

# Importar desde shared (ajustar sys.path si es necesario)
import sys
sys.path.insert(0, str(__import__("pathlib").Path(__file__).resolve().parent.parent))

from shared.events import OrderCreated
from kafka_producer import order_producer

logger = logging.getLogger(__name__)

router = APIRouter(prefix="/orders", tags=["orders"])


class CreateOrderRequest(BaseModel):
    """Solicitud para crear un nuevo pedido."""
    customer_id: str = Field(..., min_length=1)
    product_id: str = Field(..., min_length=1)
    quantity: int = Field(..., gt=0)


class OrderResponse(BaseModel):
    """Respuesta con los datos del pedido creado."""
    order_id: str
    customer_id: str
    product_id: str
    quantity: int
    status: str


# Almacenamiento en memoria (en producción sería PostgreSQL)
orders_store: dict[str, OrderResponse] = {}


@router.post(
    "",
    response_model=OrderResponse,
    status_code=status.HTTP_201_CREATED,
    summary="Crear pedido y publicar evento OrderCreated",
)
async def create_order(request: CreateOrderRequest) -> OrderResponse:
    """Crea un pedido y publica un evento OrderCreated a Kafka.

    El evento se publica al topic 'orders.created' con la key
    igual al order_id para garantizar particionamiento consistente.
    """
    order_id = uuid4()
    correlation_id = uuid4()

    # Crear el pedido
    order = OrderResponse(
        order_id=str(order_id),
        customer_id=request.customer_id,
        product_id=request.product_id,
        quantity=request.quantity,
        status="pending_stock_reservation",
    )
    orders_store[str(order_id)] = order

    # Construir y publicar el evento de dominio
    event = OrderCreated(
        order_id=order_id,
        customer_id=request.customer_id,
        product_id=request.product_id,
        quantity=request.quantity,
        correlation_id=correlation_id,
    )

    try:
        order_producer.publish_order_created(
            event_data=event.to_kafka_value(),
            order_id=str(order_id),
        )
        logger.info(
            "OrderCreated event published: event_id=%s, order_id=%s",
            event.event_id,
            order_id,
        )
    except Exception as e:
        logger.error("Failed to publish OrderCreated event: %s", e)
        # El pedido se creó pero el evento no se publicó.
        # En producción se usaría outbox pattern.
        order.status = "created_event_pending"

    return order


@router.get(
    "/{order_id}",
    response_model=OrderResponse,
    summary="Consultar pedido por ID",
)
async def get_order(order_id: str) -> OrderResponse:
    """Retorna un pedido existente por su ID."""
    if order_id not in orders_store:
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND,
            detail=f"Order '{order_id}' not found",
        )
    return orders_store[order_id]
```

4. Actualiza el archivo principal de la aplicación FastAPI. Crea o modifica `~/microservices-lab/order-service/main.py`:

```python
# ~/microservices-lab/order-service/main.py
"""Punto de entrada de OrderService con integración Kafka."""

import logging
from contextlib import asynccontextmanager

from fastapi import FastAPI

from kafka_producer import order_producer
from routes_kafka import router as orders_router

logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s [%(levelname)s] %(name)s: %(message)s",
)


@asynccontextmanager
async def lifespan(app: FastAPI):
    """Gestión del ciclo de vida: inicializa y cierra el productor Kafka."""
    logging.info("OrderService starting - Kafka producer ready")
    yield
    order_producer.close()
    logging.info("OrderService shutdown - Kafka producer closed")


app = FastAPI(
    title="OrderService",
    version="2.0.0",
    description="Servicio de pedidos con integración Kafka",
    lifespan=lifespan,
)

app.include_router(orders_router)


@app.get("/health", tags=["health"])
async def health_check():
    return {"status": "healthy", "service": "order-service"}
```

### Salida Esperada

Al iniciar el servicio, el log debe mostrar:

```
OrderService starting - Kafka producer ready
INFO:     Uvicorn running on http://0.0.0.0:8001
```

### Verificación

```bash
cd ~/microservices-lab/order-service/
uvicorn main:app --host 0.0.0.0 --port 8001 &

# Esperar 2 segundos y probar
sleep 2
curl -s -X POST http://localhost:8001/orders \
  -H "Content-Type: application/json" \
  -d '{"customer_id": "cust-001", "product_id": "prod-abc", "quantity": 3}' | python3 -m json.tool
```

Debe retornar un JSON con `status: "pending_stock_reservation"` y un `order_id` UUID.

Verifica que el mensaje llegó a Kafka:

```bash
docker exec mslab-kafka kafka-console-consumer \
  --bootstrap-server localhost:9092 \
  --topic orders.created \
  --from-beginning \
  --max-messages 1 \
  --timeout-ms 5000
```

Debe mostrar el JSON del evento `OrderCreated`.

## Paso 4: Implementar el Consumidor Kafka con Idempotencia en InventoryService

**Objetivo:** Crear un consumidor Kafka que procese eventos `OrderCreated`, implemente deduplicación con Redis y publique eventos de respuesta.

### Instrucciones

1. Instala las dependencias en InventoryService:

```bash
cd ~/microservices-lab/inventory-service/
pip install confluent-kafka==2.4.0 redis==5.0.4
```

2. Crea el módulo de deduplicación con Redis en `~/microservices-lab/inventory-service/idempotency.py`:

```python
# ~/microservices-lab/inventory-service/idempotency.py
"""Módulo de idempotencia basado en Redis para consumidores Kafka.

Usa Redis como registro de event_ids procesados con TTL de 24 horas.
Previene el procesamiento duplicado ante reentrega de mensajes.
"""

import logging
from typing import Optional

import redis

logger = logging.getLogger(__name__)

# TTL de 24 horas en segundos
IDEMPOTENCY_TTL_SECONDS = 86400


class IdempotencyGuard:
    """Guardia de idempotencia basado en Redis.

    Registra event_ids procesados exitosamente y permite verificar
    si un evento ya fue procesado antes de ejecutar la lógica de negocio.
    """

    def __init__(self, redis_url: str = "redis://localhost:6379/0"):
        self._redis: Optional[redis.Redis] = None
        self._redis_url = redis_url
        self._key_prefix = "idempotency:inventory:"

    def _get_client(self) -> redis.Redis:
        """Inicialización lazy del cliente Redis."""
        if self._redis is None:
            self._redis = redis.Redis.from_url(
                self._redis_url,
                decode_responses=True,
            )
            # Verificar conexión
            self._redis.ping()
            logger.info("Redis idempotency guard connected: %s", self._redis_url)
        return self._redis

    def is_duplicate(self, event_id: str) -> bool:
        """Verifica si un evento ya fue procesado.

        Args:
            event_id: UUID del evento a verificar.

        Returns:
            True si el evento ya fue procesado (duplicado), False si es nuevo.
        """
        client = self._get_client()
        key = f"{self._key_prefix}{event_id}"
        exists = client.exists(key)
        if exists:
            logger.warning("Evento duplicado detectado: event_id=%s", event_id)
        return bool(exists)

    def mark_processed(self, event_id: str) -> None:
        """Marca un evento como procesado exitosamente.

        Args:
            event_id: UUID del evento a marcar.
        """
        client = self._get_client()
        key = f"{self._key_prefix}{event_id}"
        client.setex(key, IDEMPOTENCY_TTL_SECONDS, "processed")
        logger.info(
            "Evento marcado como procesado: event_id=%s (TTL=%ds)",
            event_id,
            IDEMPOTENCY_TTL_SECONDS,
        )

    def close(self) -> None:
        """Cierra la conexión Redis."""
        if self._redis is not None:
            self._redis.close()
            self._redis = None


# Instancia singleton
idempotency_guard = IdempotencyGuard()
```

3. Crea el consumidor Kafka en `~/microservices-lab/inventory-service/kafka_consumer.py`:

```python
# ~/microservices-lab/inventory-service/kafka_consumer.py
"""Consumidor Kafka para InventoryService.

Consume eventos OrderCreated del topic 'orders.created',
verifica idempotencia, procesa la reserva de stock y publica
eventos de respuesta (StockReserved o StockReservationFailed).
"""

import json
import logging
import threading
from typing import Optional

from confluent_kafka import Consumer, KafkaError, KafkaException, Producer

from idempotency import idempotency_guard

import sys
sys.path.insert(0, str(__import__("pathlib").Path(__file__).resolve().parent.parent))

from shared.events import OrderCreated, StockReserved, StockReservationFailed

logger = logging.getLogger(__name__)


# Simulación de inventario en memoria
# En producción esto sería PostgreSQL
inventory_stock: dict[str, int] = {
    "prod-abc": 100,
    "prod-def": 50,
    "prod-ghi": 25,
    "prod-xyz": 0,
}


class InventoryEventConsumer:
    """Consumidor de eventos para el servicio de inventario.

    Procesa eventos OrderCreated e intenta reservar stock.
    Implementa idempotencia mediante Redis y publica eventos de respuesta.
    """

    def __init__(
        self,
        bootstrap_servers: str = "localhost:29092",
        group_id: str = "inventory-service-group",
    ):
        self._consumer_config = {
            "bootstrap.servers": bootstrap_servers,
            "group.id": group_id,
            "auto.offset.reset": "earliest",
            "enable.auto.commit": False,  # Commit manual para control preciso
            "max.poll.interval.ms": 300000,
        }
        self._producer_config = {
            "bootstrap.servers": bootstrap_servers,
            "client.id": "inventory-service-producer",
            "acks": "all",
            "enable.idempotence": True,
        }
        self._consumer: Optional[Consumer] = None
        self._producer: Optional[Producer] = None
        self._running = False
        self._thread: Optional[threading.Thread] = None

    def _get_producer(self) -> Producer:
        """Inicializa el productor para eventos de respuesta."""
        if self._producer is None:
            self._producer = Producer(self._producer_config)
        return self._producer

    def _process_order_created(self, event: OrderCreated) -> None:
        """Procesa un evento OrderCreated: intenta reservar stock.

        Si el stock es suficiente, reserva y publica StockReserved.
        Si no, publica StockReservationFailed.
        La idempotencia se verifica ANTES de esta función.
        """
        product_id = event.product_id
        quantity = event.quantity
        current_stock = inventory_stock.get(product_id, 0)

        producer = self._get_producer()

        if current_stock >= quantity:
            # Reservar stock
            inventory_stock[product_id] = current_stock - quantity
            remaining = inventory_stock[product_id]

            response_event = StockReserved(
                order_id=event.order_id,
                product_id=product_id,
                quantity_reserved=quantity,
                remaining_stock=remaining,
                correlation_id=event.correlation_id,
            )
            producer.produce(
                topic="inventory.stock-reserved",
                key=str(event.order_id).encode("utf-8"),
                value=response_event.to_kafka_value(),
            )
            producer.flush(timeout=5.0)
            logger.info(
                "Stock reservado: order_id=%s, product=%s, qty=%d, remaining=%d",
                event.order_id,
                product_id,
                quantity,
                remaining,
            )
        else:
            # Stock insuficiente
            response_event = StockReservationFailed(
                order_id=event.order_id,
                product_id=product_id,
                quantity_requested=quantity,
                available_stock=current_stock,
                reason=f"Insufficient stock: requested {quantity}, available {current_stock}",
                correlation_id=event.correlation_id,
            )
            producer.produce(
                topic="inventory.stock-failed",
                key=str(event.order_id).encode("utf-8"),
                value=response_event.to_kafka_value(),
            )
            producer.flush(timeout=5.0)
            logger.warning(
                "Reserva fallida: order_id=%s, product=%s, requested=%d, available=%d",
                event.order_id,
                product_id,
                quantity,
                current_stock,
            )

    def _consume_loop(self) -> None:
        """Bucle principal del consumidor."""
        self._consumer = Consumer(self._consumer_config)
        self._consumer.subscribe(["orders.created"])
        logger.info("Consumer subscribed to 'orders.created'")

        while self._running:
            msg = self._consumer.poll(timeout=1.0)

            if msg is None:
                continue

            if msg.error():
                if msg.error().code() == KafkaError._PARTITION_EOF:
                    continue
                logger.error("Consumer error: %s", msg.error())
                continue

            try:
                # Deserializar el evento
                event = OrderCreated.from_kafka_value(msg.value())
                event_id_str = str(event.event_id)

                logger.info(
                    "Evento recibido: event_id=%s, order_id=%s, partition=%d, offset=%d",
                    event_id_str,
                    event.order_id,
                    msg.partition(),
                    msg.offset(),
                )

                # Verificar idempotencia ANTES de procesar
                if idempotency_guard.is_duplicate(event_id_str):
                    logger.info("Evento duplicado ignorado: event_id=%s", event_id_str)
                    # Commit del offset para no re-procesarlo
                    self._consumer.commit(msg)
                    continue

                # Procesar el evento
                self._process_order_created(event)

                # Marcar como procesado en Redis
                idempotency_guard.mark_processed(event_id_str)

                # Commit manual del offset
                self._consumer.commit(msg)

            except Exception as e:
                logger.error(
                    "Error procesando mensaje (partition=%d, offset=%d): %s",
                    msg.partition(),
                    msg.offset(),
                    e,
                )
                # No hacer commit: el mensaje se reentregará

    def start(self) -> None:
        """Inicia el consumidor en un hilo separado."""
        if self._running:
            return
        self._running = True
        self._thread = threading.Thread(target=self._consume_loop, daemon=True)
        self._thread.start()
        logger.info("Inventory consumer started in background thread")

    def stop(self) -> None:
        """Detiene el consumidor de forma graceful."""
        self._running = False
        if self._thread is not None:
            self._thread.join(timeout=10.0)
        if self._consumer is not None:
            self._consumer.close()
            self._consumer = None
        if self._producer is not None:
            self._producer.flush(timeout=5.0)
            self._producer = None
        idempotency_guard.close()
        logger.info("Inventory consumer stopped")


# Instancia singleton
inventory_consumer = InventoryEventConsumer()
```

4. Crea el archivo principal de InventoryService en `~/microservices-lab/inventory-service/main.py`:

```python
# ~/microservices-lab/inventory-service/main.py
"""Punto de entrada de InventoryService con consumidor Kafka."""

import logging
from contextlib import asynccontextmanager

from fastapi import FastAPI

from kafka_consumer import inventory_consumer, inventory_stock

logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s [%(levelname)s] %(name)s: %(message)s",
)


@asynccontextmanager
async def lifespan(app: FastAPI):
    """Ciclo de vida: inicia y detiene el consumidor Kafka."""
    inventory_consumer.start()
    logging.info("InventoryService started - Kafka consumer active")
    yield
    inventory_consumer.stop()
    logging.info("InventoryService shutdown - Kafka consumer stopped")


app = FastAPI(
    title="InventoryService",
    version="2.0.0",
    description="Servicio de inventario con consumo de eventos Kafka",
    lifespan=lifespan,
)


@app.get("/health", tags=["health"])
async def health_check():
    return {"status": "healthy", "service": "inventory-service"}


@app.get("/inventory/{product_id}", tags=["inventory"])
async def get_stock(product_id: str):
    """Consulta el stock actual de un producto."""
    stock = inventory_stock.get(product_id)
    if stock is None:
        return {"product_id": product_id, "stock": 0, "exists": False}
    return {"product_id": product_id, "stock": stock, "exists": True}
```

### Salida Esperada

Al iniciar InventoryService:

```
InventoryService started - Kafka consumer active
INFO:     Uvicorn running on http://0.0.0.0:8002
Consumer subscribed to 'orders.created'
```

### Verificación

```bash
cd ~/microservices-lab/inventory-service/
uvicorn main:app --host 0.0.0.0 --port 8002 &
sleep 3

# Verificar health
curl -s http://localhost:8002/health | python3 -m json.tool

# Verificar stock inicial
curl -s http://localhost:8002/inventory/prod-abc | python3 -m json.tool
```

Debe mostrar `"stock": 100` para `prod-abc`.

## Paso 5: Validar el Flujo Completo de Eventos

**Objetivo:** Ejecutar el flujo end-to-end: crear un pedido → evento publicado → stock reservado → evento de respuesta.

### Instrucciones

1. Asegúrate de que ambos servicios estén corriendo (OrderService en 8001, InventoryService en 8002).

2. Crea un pedido que dispare el flujo completo:

```bash
curl -s -X POST http://localhost:8001/orders \
  -H "Content-Type: application/json" \
  -d '{"customer_id": "cust-001", "product_id": "prod-abc", "quantity": 5}' | python3 -m json.tool
```

3. Espera 3 segundos para que el consumidor procese el evento:

```bash
sleep 3
```

4. Verifica que el stock se redujo:

```bash
curl -s http://localhost:8002/inventory/prod-abc | python3 -m json.tool
```

5. Verifica el evento de respuesta en el topic `inventory.stock-reserved`:

```bash
docker exec mslab-kafka kafka-console-consumer \
  --bootstrap-server localhost:9092 \
  --topic inventory.stock-reserved \
  --from-beginning \
  --max-messages 1 \
  --timeout-ms 5000
```

6. Prueba un caso de stock insuficiente:

```bash
curl -s -X POST http://localhost:8001/orders \
  -H "Content-Type: application/json" \
  -d '{"customer_id": "cust-002", "product_id": "prod-xyz", "quantity": 10}' | python3 -m json.tool

sleep 3

docker exec mslab-kafka kafka-console-consumer \
  --bootstrap-server localhost:9092 \
  --topic inventory.stock-failed \
  --from-beginning \
  --max-messages 1 \
  --timeout-ms 5000
```

### Salida Esperada

Stock después de la reserva:

```json
{
    "product_id": "prod-abc",
    "stock": 95,
    "exists": true
}
```

Evento `StockReservationFailed` para `prod-xyz`:

```json
{
    "event_id": "...",
    "event_type": "StockReservationFailed",
    "reason": "Insufficient stock: requested 10, available 0",
    ...
}
```

### Verificación

El stock de `prod-abc` debe haber disminuido de 100 a 95, confirmando que el evento fue procesado correctamente.

## Paso 6: Validar Idempotencia bajo Reentrega de Mensajes

**Objetivo:** Demostrar que el mecanismo de deduplicación con Redis previene el procesamiento duplicado cuando un mensaje se reentrega.

### Instrucciones

1. Crea el script de prueba de idempotencia en `~/microservices-lab/tests/test_idempotency.py`:

```python
# ~/microservices-lab/tests/test_idempotency.py
"""Script de prueba de idempotencia.

Simula la reentrega de un mismo evento OrderCreated múltiples veces
y verifica que el stock solo se reduce una vez.
"""

import json
import sys
import time
from pathlib import Path
from uuid import uuid4

from confluent_kafka import Producer
import redis
import requests

# Agregar shared al path
sys.path.insert(0, str(Path(__file__).resolve().parent.parent))
from shared.events import OrderCreated


KAFKA_BOOTSTRAP = "localhost:29092"
REDIS_URL = "redis://localhost:6379/0"
INVENTORY_URL = "http://localhost:8002"


def get_stock(product_id: str) -> int:
    """Consulta el stock actual de un producto."""
    resp = requests.get(f"{INVENTORY_URL}/inventory/{product_id}")
    resp.raise_for_status()
    return resp.json()["stock"]


def produce_event_directly(event: OrderCreated, topic: str = "orders.created") -> None:
    """Publica un evento directamente al topic Kafka (bypass del servicio)."""
    producer = Producer({"bootstrap.servers": KAFKA_BOOTSTRAP})
    producer.produce(
        topic=topic,
        key=str(event.order_id).encode("utf-8"),
        value=event.to_kafka_value(),
    )
    producer.flush(timeout=5.0)


def main():
    """Prueba principal de idempotencia."""
    print("=" * 60)
    print("PRUEBA DE IDEMPOTENCIA - REENTREGA DE MENSAJES")
    print("=" * 60)

    product_id = "prod-def"
    quantity = 5

    # 1. Capturar stock inicial
    stock_before = get_stock(product_id)
    print(f"\n[1] Stock inicial de '{product_id}': {stock_before}")

    # 2. Crear un evento con un event_id fijo
    fixed_event_id = uuid4()
    order_id = uuid4()
    correlation_id = uuid4()

    event = OrderCreated(
        event_id=fixed_event_id,
        order_id=order_id,
        customer_id="test-customer",
        product_id=product_id,
        quantity=quantity,
        correlation_id=correlation_id,
    )

    print(f"\n[2] Evento creado: event_id={fixed_event_id}")
    print(f"    order_id={order_id}, quantity={quantity}")

    # 3. Publicar el mismo evento 3 veces (simulando reentrega)
    print("\n[3] Publicando el MISMO evento 3 veces (simulando reentrega)...")
    for i in range(3):
        produce_event_directly(event)
        print(f"    Envío #{i+1} completado")
        time.sleep(0.5)

    # 4. Esperar procesamiento
    print("\n[4] Esperando procesamiento (5 segundos)...")
    time.sleep(5)

    # 5. Verificar stock final
    stock_after = get_stock(product_id)
    print(f"\n[5] Stock final de '{product_id}': {stock_after}")

    # 6. Calcular reducción
    reduction = stock_before - stock_after
    expected_reduction = quantity  # Solo debe reducirse UNA vez

    print(f"\n[6] Reducción de stock: {reduction} unidades")
    print(f"    Reducción esperada: {expected_reduction} unidades")

    # 7. Verificar resultado
    print("\n" + "=" * 60)
    if reduction == expected_reduction:
        print("✅ PRUEBA EXITOSA: La idempotencia funciona correctamente.")
        print(f"   El evento se procesó exactamente 1 vez a pesar de 3 entregas.")
    else:
        print("❌ PRUEBA FALLIDA: La idempotencia NO funcionó.")
        print(f"   Se esperaba reducción de {expected_reduction}, pero fue {reduction}.")
        sys.exit(1)
    print("=" * 60)

    # 8. Verificar en Redis que el event_id está registrado
    r = redis.Redis.from_url(REDIS_URL, decode_responses=True)
    key = f"idempotency:inventory:{fixed_event_id}"
    value = r.get(key)
    ttl = r.ttl(key)
    print(f"\n[7] Redis verification:")
    print(f"    Key: {key}")
    print(f"    Value: {value}")
    print(f"    TTL remaining: {ttl} seconds")
    r.close()


if __name__ == "__main__":
    main()
```

2. Instala las dependencias de prueba:

```bash
pip install requests pytest==8.2.1 pytest-asyncio==0.23.7
```

3. Ejecuta la prueba de idempotencia:

```bash
cd ~/microservices-lab/
python3 tests/test_idempotency.py
```

### Salida Esperada

```
============================================================
PRUEBA DE IDEMPOTENCIA - REENTREGA DE MENSAJES
============================================================

[1] Stock inicial de 'prod-def': 50

[2] Evento creado: event_id=<uuid>
    order_id=<uuid>, quantity=5

[3] Publicando el MISMO evento 3 veces (simulando reentrega)...
    Envío #1 completado
    Envío #2 completado
    Envío #3 completado

[4] Esperando procesamiento (5 segundos)...

[5] Stock final de 'prod-def': 45

[6] Reducción de stock: 5 unidades
    Reducción esperada: 5 unidades

============================================================
✅ PRUEBA EXITOSA: La idempotencia funciona correctamente.
   El evento se procesó exactamente 1 vez a pesar de 3 entregas.
============================================================

[7] Redis verification:
    Key: idempotency:inventory:<uuid>
    Value: processed
    TTL remaining: 86399 seconds
```

### Verificación

La reducción de stock debe ser exactamente `5` (no `15`), demostrando que las entregas duplicadas fueron detectadas y descartadas.

## Paso 7: Validar Particionamiento por order_id

**Objetivo:** Confirmar que eventos del mismo `order_id` siempre llegan a la misma partición, garantizando orden por entidad.

### Instrucciones

1. Crea el script de verificación de particionamiento en `~/microservices-lab/tests/test_partitioning.py`:

```python
# ~/microservices-lab/tests/test_partitioning.py
"""Verifica que el particionamiento por order_id es consistente."""

import sys
from pathlib import Path
from uuid import uuid4

from confluent_kafka import Producer
from confluent_kafka.admin import AdminClient, TopicMetadata

sys.path.insert(0, str(Path(__file__).resolve().parent.parent))
from shared.events import OrderCreated

KAFKA_BOOTSTRAP = "localhost:29092"


def get_partition_for_key(key: bytes, num_partitions: int) -> int:
    """Calcula la partición usando el mismo algoritmo que Kafka (murmur2)."""
    # confluent-kafka usa murmur2 por defecto para el particionamiento
    # Aquí verificamos empíricamente
    pass


def main():
    print("=" * 60)
    print("PRUEBA DE PARTICIONAMIENTO POR ORDER_ID")
    print("=" * 60)

    # Crear un producer con callback de delivery para capturar la partición
    results = {}

    def delivery_cb(err, msg):
        if err is None:
            key = msg.key().decode("utf-8")
            if key not in results:
                results[key] = []
            results[key].append(msg.partition())

    producer = Producer({"bootstrap.servers": KAFKA_BOOTSTRAP})

    # Generar 3 order_ids diferentes, cada uno con 5 eventos
    order_ids = [str(uuid4()) for _ in range(3)]

    print(f"\n[1] Publicando 5 eventos para cada uno de 3 order_ids...")
    for order_id in order_ids:
        for i in range(5):
            event = OrderCreated(
                order_id=order_id,
                customer_id=f"cust-{i}",
                product_id="prod-abc",
                quantity=1,
                correlation_id=uuid4(),
            )
            producer.produce(
                topic="orders.created",
                key=order_id.encode("utf-8"),
                value=event.to_kafka_value(),
                callback=delivery_cb,
            )

    producer.flush(timeout=10.0)

    print("\n[2] Resultados de particionamiento:")
    all_consistent = True
    for order_id, partitions in results.items():
        unique_partitions = set(partitions)
        consistent = len(unique_partitions) == 1
        status = "✅" if consistent else "❌"
        print(f"    {status} order_id={order_id[:8]}... -> particiones: {partitions}")
        if not consistent:
            all_consistent = False

    print("\n" + "=" * 60)
    if all_consistent:
        print("✅ PRUEBA EXITOSA: Todos los eventos del mismo order_id")
        print("   van a la misma partición (orden garantizado por entidad).")
    else:
        print("❌ PRUEBA FALLIDA: Inconsistencia en el particionamiento.")
        sys.exit(1)
    print("=" * 60)


if __name__ == "__main__":
    main()
```

2. Ejecuta la prueba:

```bash
cd ~/microservices-lab/
python3 tests/test_partitioning.py
```

### Salida Esperada

```
============================================================
PRUEBA DE PARTICIONAMIENTO POR ORDER_ID
============================================================

[1] Publicando 5 eventos para cada uno de 3 order_ids...

[2] Resultados de particionamiento:
    ✅ order_id=a1b2c3d4... -> particiones: [1, 1, 1, 1, 1]
    ✅ order_id=e5f6g7h8... -> particiones: [0, 0, 0, 0, 0]
    ✅ order_id=i9j0k1l2... -> particiones: [2, 2, 2, 2, 2]

============================================================
✅ PRUEBA EXITOSA: Todos los eventos del mismo order_id
   van a la misma partición (orden garantizado por entidad).
============================================================
```

### Verificación

Cada `order_id` debe mostrar siempre la misma partición en todas sus entregas, confirmando que el particionamiento por key funciona correctamente.

## Validación y Pruebas

Ejecuta la validación completa del laboratorio con este script integrador:

```bash
# ~/microservices-lab/tests/validate_lab02.sh
#!/bin/bash
set -e

echo "============================================"
echo "  VALIDACIÓN COMPLETA - LAB 02"
echo "============================================"

PASS=0
FAIL=0

# Test 1: Servicios de infraestructura activos
echo -e "\n[TEST 1] Verificando infraestructura..."
if docker exec mslab-kafka kafka-topics --list --bootstrap-server localhost:9092 | grep -q "orders.created"; then
    echo "  ✅ Kafka topics existen"
    ((PASS++))
else
    echo "  ❌ Kafka topics no encontrados"
    ((FAIL++))
fi

if docker exec mslab-redis redis-cli ping | grep -q "PONG"; then
    echo "  ✅ Redis operativo"
    ((PASS++))
else
    echo "  ❌ Redis no responde"
    ((FAIL++))
fi

# Test 2: Servicios REST operativos
echo -e "\n[TEST 2] Verificando servicios REST..."
if curl -sf http://localhost:8001/health > /dev/null; then
    echo "  ✅ OrderService healthy (puerto 8001)"
    ((PASS++))
else
    echo "  ❌ OrderService no responde"
    ((FAIL++))
fi

if curl -sf http://localhost:8002/health > /dev/null; then
    echo "  ✅ InventoryService healthy (puerto 8002)"
    ((PASS++))
else
    echo "  ❌ InventoryService no responde"
    ((FAIL++))
fi

# Test 3: Flujo end-to-end
echo -e "\n[TEST 3] Flujo end-to-end (crear pedido → reservar stock)..."
STOCK_BEFORE=$(curl -s http://localhost:8002/inventory/prod-ghi | python3 -c "import sys,json; print(json.load(sys.stdin)['stock'])")
echo "  Stock antes: $STOCK_BEFORE"

RESPONSE=$(curl -s -X POST http://localhost:8001/orders \
  -H "Content-Type: application/json" \
  -d '{"customer_id": "val-cust", "product_id": "prod-ghi", "quantity": 2}')
echo "  Pedido creado: $(echo $RESPONSE | python3 -c "import sys,json; print(json.load(sys.stdin)['order_id'][:8])...")

sleep 4

STOCK_AFTER=$(curl -s http://localhost:8002/inventory/prod-ghi | python3 -c "import sys,json; print(json.load(sys.stdin)['stock'])")
echo "  Stock después: $STOCK_AFTER"

EXPECTED=$((STOCK_BEFORE - 2))
if [ "$STOCK_AFTER" -eq "$EXPECTED" ]; then
    echo "  ✅ Stock reducido correctamente ($STOCK_BEFORE → $STOCK_AFTER)"
    ((PASS++))
else
    echo "  ❌ Stock incorrecto: esperado $EXPECTED, obtenido $STOCK_AFTER"
    ((FAIL++))
fi

# Test 4: Esquemas de eventos
echo -e "\n[TEST 4] Verificando esquemas de eventos..."
if python3 -c "
from shared.events import OrderCreated, StockReserved, StockReservationFailed
from uuid import uuid4
e = OrderCreated(order_id=uuid4(), customer_id='x', product_id='y', quantity=1, correlation_id=uuid4())
assert e.event_version == '1.0.0'
assert e.source_service == 'order-service'
data = e.to_kafka_value()
e2 = OrderCreated.from_kafka_value(data)
assert e.event_id == e2.event_id
print('OK')
" 2>/dev/null | grep -q "OK"; then
    echo "  ✅ Esquemas de eventos válidos con serialización/deserialización"
    ((PASS++))
else
    echo "  ❌ Error en esquemas de eventos"
    ((FAIL++))
fi

# Resumen
echo -e "\n============================================"
echo "  RESULTADOS: $PASS pasaron, $FAIL fallaron"
echo "============================================"

if [ $FAIL -eq 0 ]; then
    echo "  🎉 ¡Todos los tests pasaron!"
else
    echo "  ⚠️  Hay tests fallidos. Revisa los pasos anteriores."
    exit 1
fi
```

Ejecútalo:

```bash
chmod +x ~/microservices-lab/tests/validate_lab02.sh
~/microservices-lab/tests/validate_lab02.sh
```

## Solución de Problemas

### Problema 1: El consumidor no procesa mensajes (stock no cambia)

**Síntomas:** Se crea un pedido exitosamente, el evento aparece en el topic `orders.created` (verificable con `kafka-console-consumer`), pero el stock en InventoryService no se reduce.

**Causa:** El consumer group no está recibiendo mensajes. Esto ocurre típicamente cuando:
- El consumidor se inició antes de que el topic existiera y `auto.offset.reset` no está configurado correctamente.
- El `bootstrap.servers` del consumidor apunta a `localhost:9092` en lugar de `localhost:29092` (puerto externo del contenedor).

**Solución:**

```bash
# 1. Verificar que el consumidor está conectado al grupo correcto
docker exec mslab-kafka kafka-consumer-groups \
  --bootstrap-server localhost:9092 \
  --group inventory-service-group \
  --describe

# 2. Si el grupo no aparece o tiene lag, reiniciar InventoryService
# Detener el proceso uvicorn de InventoryService y reiniciarlo

# 3. Verificar la conectividad desde el host
python3 -c "
from confluent_kafka import Consumer
c = Consumer({'bootstrap.servers': 'localhost:29092', 'group.id': 'test-group', 'auto.offset.reset': 'earliest'})
c.subscribe(['orders.created'])
msg = c.poll(5.0)
print('Mensaje recibido' if msg and not msg.error() else 'Sin mensajes o error')
c.close()
"

# 4. Si el problema persiste, resetear los offsets del grupo
docker exec mslab-kafka kafka-consumer-groups \
  --bootstrap-server localhost:9092 \
  --group inventory-service-group \
  --topic orders.created \
  --reset-offsets \
  --to-earliest \
  --execute
```

### Problema 2: Redis rechaza conexiones (idempotencia no funciona)

**Síntomas:** Los logs del consumidor muestran `ConnectionError: Error connecting to Redis` o `redis.exceptions.ConnectionError`. Los eventos se procesan múltiples veces generando reducciones de stock duplicadas.

**Causa:** El contenedor Redis no está en la misma red Docker que el proceso del host, o el puerto 6379 no está expuesto. También puede ocurrir si Redis se quedó sin memoria por el límite de `maxmemory`.

**Solución:**

```bash
# 1. Verificar que Redis está corriendo y el puerto expuesto
docker ps --filter name=mslab-redis
docker exec mslab-redis redis-cli ping
# Debe responder: PONG

# 2. Verificar conectividad desde el host
python3 -c "
import redis
r = redis.Redis(host='localhost', port=6379, decode_responses=True)
r.ping()
print('Redis conectado correctamente')
r.set('test-key', 'test-value', ex=10)
print(f'Test value: {r.get(\"test-key\")}')
r.close()
"

# 3. Si Redis reporta OOM, verificar uso de memoria
docker exec mslab-redis redis-cli INFO memory | grep used_memory_human

# 4. Si está lleno, limpiar las keys de idempotencia antiguas
docker exec mslab-redis redis-cli KEYS "idempotency:*" | head -5
# Para limpiar todo (solo en desarrollo):
docker exec mslab-redis redis-cli FLUSHDB

# 5. Reiniciar Redis si es necesario
docker restart mslab-redis
sleep 2
docker exec mslab-redis redis-cli ping
```

## Limpieza

Detén todos los servicios y contenedores cuando finalices el laboratorio:

```bash
# Detener los procesos uvicorn en background
pkill -f "uvicorn.*8001" 2>/dev/null || true
pkill -f "uvicorn.*8002" 2>/dev/null || true

# Detener y eliminar los contenedores de infraestructura
cd ~/microservices-lab/
docker compose -f docker-compose.kafka.yml down -v

# Verificar que no quedan contenedores
docker ps --filter name=mslab- --format "{{.Names}}"
# No debe mostrar nada

# (Opcional) Limpiar imágenes descargadas para liberar espacio
# docker rmi confluentinc/cp-kafka:7.6.1 confluentinc/cp-zookeeper:7.6.1 redis:7.2.5-alpine
```

## Resumen

En este laboratorio implementaste exitosamente:

| Componente | Descripción |
|------------|-------------|
| **Infraestructura Kafka** | Broker con Zookeeper, 3 topics con 3 particiones cada uno |
| **Esquemas de eventos** | Modelos Pydantic v2 con UUID v4, versionado y serialización JSON |
| **Productor (OrderService)** | Publicación de `OrderCreated` con `acks=all` e idempotencia del productor |
| **Consumidor (InventoryService)** | Procesamiento de eventos con commit manual de offsets |
| **Idempotencia** | Deduplicación con Redis (TTL 24h) que previene procesamiento duplicado |
| **Particionamiento** | Key-based partitioning por `order_id` garantizando orden por entidad |

**Conceptos clave aplicados:**

- La mensajería basada en eventos desacopla temporalmente los servicios (OrderService no espera respuesta de InventoryService)
- La idempotencia es responsabilidad del consumidor, no del broker
- El particionamiento por key garantiza orden FIFO dentro de una misma entidad
- El commit manual de offsets permite control preciso sobre cuándo se confirma el procesamiento

### Recursos Adicionales

- [Documentación confluent-kafka-python](https://docs.confluent.io/kafka-clients/python/current/overview.html)
- [Kafka Design: Log Compaction & Idempotent Producers](https://kafka.apache.org/documentation/#design)
- [Redis SET con TTL para deduplicación](https://redis.io/commands/setex/)
- [Pydantic v2 Model Serialization](https://docs.pydantic.dev/latest/concepts/serialization/)
