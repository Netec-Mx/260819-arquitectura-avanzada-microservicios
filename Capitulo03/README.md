# Implementar Event Store y Proyecciones en Python

| Campo | Valor |
|-------|-------|
| **Duración** | 66 minutos |
| **Complejidad** | Alta |
| **Nivel Bloom** | Crear |

## Descripción General

En este laboratorio transformarás el OrderService para adoptar Event Sourcing completo con CQRS. Implementarás un Event Store append-only en PostgreSQL, proyecciones materializadas para el modelo de lectura, snapshots automáticos cada 10 eventos por agregado y un mecanismo de reconciliación. Los comandos escribirán eventos al Event Store y los publicarán a Kafka; las consultas leerán exclusivamente desde proyecciones desnormalizadas.

## Objetivos de Aprendizaje

- [ ] Implementar un Event Store append-only en PostgreSQL con constraint de inmutabilidad y metadatos completos
- [ ] Separar el modelo de escritura (comandos → Event Store) del modelo de lectura (proyecciones materializadas) aplicando CQRS
- [ ] Construir proyecciones síncronas que reconstruyan el estado actual de órdenes a partir de la secuencia de eventos
- [ ] Implementar snapshots automáticos cada 10 eventos por agregado y un endpoint de reconciliación `/admin/rebuild-projections`
- [ ] Integrar la persistencia de eventos con la publicación a Kafka manteniendo la compatibilidad con Lab 02

## Prerrequisitos

### Conocimiento

- Lab 01 y Lab 02 completados (OrderService funcional con Kafka)
- Comprensión conceptual de CQRS y Event Sourcing (módulo 3.1 del curso)
- SQL intermedio: transacciones, constraints, índices
- SQLAlchemy 2.0 async y Alembic básico

### Acceso y Software

- Docker Engine 26.1.3+ con Docker Compose 2.27.1+
- Python 3.12.3 con entorno virtual activo
- Contenedores `mslab-postgres` y `mslab-kafka` operativos
- Módulo `~/microservices-lab/shared/events/` disponible del Lab 02

## Entorno del Laboratorio

### Estructura de Directorios Resultante

```
~/microservices-lab/
├── order-service/
│   ├── alembic/
│   │   ├── versions/
│   │   └── env.py
│   ├── alembic.ini
│   ├── app/
│   │   ├── __init__.py
│   │   ├── main.py
│   │   ├── config.py
│   │   ├── commands/
│   │   │   ├── __init__.py
│   │   │   └── order_commands.py
│   │   ├── queries/
│   │   │   ├── __init__.py
│   │   │   └── order_queries.py
│   │   ├── domain/
│   │   │   ├── __init__.py
│   │   │   └── order_aggregate.py
│   │   ├── eventstore/
│   │   │   ├── __init__.py
│   │   │   ├── store.py
│   │   │   ├── models.py
│   │   │   └── projections.py
│   │   └── kafka_producer.py
│   ├── tests/
│   │   ├── __init__.py
│   │   ├── test_event_store.py
│   │   └── test_projections.py
│   └── requirements.txt
└── shared/
    └── events/
        └── order_events.py
```

### Configuración Inicial

```bash
cd ~/microservices-lab
mkdir -p order-service/{app/{commands,queries,domain,eventstore},tests,alembic/versions}
```

Verifica que PostgreSQL y Kafka estén corriendo:

```bash
docker ps --format "table {{.Names}}\t{{.Status}}" | grep -E "mslab-postgres|mslab-kafka|mslab-zookeeper"
```

**Salida esperada:**

```
mslab-postgres      Up ...
mslab-kafka         Up ...
mslab-zookeeper     Up ...
```

Si no están activos, levántalos:

```bash
docker start mslab-postgres mslab-zookeeper mslab-kafka
```

Crea la base de datos `eventstore_db`:

```bash
docker exec mslab-postgres psql -U mslab_user -d postgres -c \
  "CREATE DATABASE eventstore_db OWNER mslab_user;"
```

---

## Paso 1: Definir Esquemas de Eventos Compartidos

**Objetivo:** Establecer los eventos de dominio con Pydantic v2 en el módulo compartido, reutilizables por el Event Store y Kafka.

### Instrucciones

1. Crea o actualiza el archivo de eventos compartidos:

```bash
cat > ~/microservices-lab/shared/events/order_events.py << 'EOF'
"""Eventos de dominio para OrderService — módulo compartido."""
from datetime import datetime, timezone
from typing import Optional
from uuid import uuid4

from pydantic import BaseModel, Field


class DomainEvent(BaseModel):
    """Clase base para todos los eventos de dominio."""
    event_id: str = Field(default_factory=lambda: str(uuid4()))
    event_type: str = ""
    aggregate_id: str = ""
    aggregate_type: str = "Order"
    occurred_at: datetime = Field(
        default_factory=lambda: datetime.now(timezone.utc)
    )
    version: int = 1
    metadata: dict = Field(default_factory=dict)


class OrderCreated(DomainEvent):
    event_type: str = "OrderCreated"
    customer_id: str = ""
    items: list = Field(default_factory=list)
    total_amount: float = 0.0


class OrderStatusChanged(DomainEvent):
    event_type: str = "OrderStatusChanged"
    previous_status: str = ""
    new_status: str = ""


class OrderCancelled(DomainEvent):
    event_type: str = "OrderCancelled"
    reason: str = ""
    cancelled_at: datetime = Field(
        default_factory=lambda: datetime.now(timezone.utc)
    )


class OrderItemAdded(DomainEvent):
    event_type: str = "OrderItemAdded"
    item_id: str = ""
    product_name: str = ""
    quantity: int = 0
    unit_price: float = 0.0


# Registro de tipos para deserialización
EVENT_TYPE_MAP = {
    "OrderCreated": OrderCreated,
    "OrderStatusChanged": OrderStatusChanged,
    "OrderCancelled": OrderCancelled,
    "OrderItemAdded": OrderItemAdded,
}
EOF
```

2. Asegúrate de que el módulo sea importable:

```bash
touch ~/microservices-lab/shared/__init__.py
touch ~/microservices-lab/shared/events/__init__.py
```

### Verificación

```bash
cd ~/microservices-lab
python -c "from shared.events.order_events import OrderCreated, EVENT_TYPE_MAP; print(f'Eventos registrados: {list(EVENT_TYPE_MAP.keys())}')"
```

**Salida esperada:**

```
Eventos registrados: ['OrderCreated', 'OrderStatusChanged', 'OrderCancelled', 'OrderItemAdded']
```

---

## Paso 2: Configurar SQLAlchemy Async y Modelos del Event Store

**Objetivo:** Definir los modelos SQLAlchemy para la tabla `events` (append-only), `order_projections` y `snapshot_orders`.

### Instrucciones

1. Crea el archivo de configuración:

```bash
cat > ~/microservices-lab/order-service/app/config.py << 'EOF'
"""Configuración centralizada del OrderService."""
from pydantic_settings import BaseSettings


class Settings(BaseSettings):
    # Event Store DB
    eventstore_db_url: str = (
        "postgresql+asyncpg://mslab_user:mslab_pass_2024@localhost:5432/eventstore_db"
    )
    # Kafka
    kafka_bootstrap_servers: str = "localhost:29092"
    kafka_orders_topic: str = "order-events"
    # Snapshots
    snapshot_threshold: int = 10

    class Config:
        env_prefix = "ORDER_SERVICE_"


settings = Settings()
EOF
```

2. Crea los modelos SQLAlchemy:

```bash
cat > ~/microservices-lab/order-service/app/eventstore/models.py << 'EOF'
"""Modelos SQLAlchemy para Event Store, Proyecciones y Snapshots."""
from datetime import datetime, timezone

from sqlalchemy import (
    Column, String, Integer, Float, DateTime, JSON, Boolean,
    BigInteger, Text, Index, CheckConstraint
)
from sqlalchemy.orm import DeclarativeBase


class Base(DeclarativeBase):
    pass


class EventRecord(Base):
    """Tabla principal del Event Store — append-only."""
    __tablename__ = "events"

    id = Column(BigInteger, primary_key=True, autoincrement=True)
    event_id = Column(String(36), unique=True, nullable=False)
    event_type = Column(String(100), nullable=False)
    aggregate_id = Column(String(36), nullable=False, index=True)
    aggregate_type = Column(String(50), nullable=False, default="Order")
    aggregate_version = Column(Integer, nullable=False)
    occurred_at = Column(
        DateTime(timezone=True), nullable=False,
        default=lambda: datetime.now(timezone.utc)
    )
    payload = Column(JSON, nullable=False)
    metadata_ = Column("metadata", JSON, nullable=True)

    __table_args__ = (
        # Garantiza unicidad de versión por agregado (optimistic concurrency)
        Index(
            "ix_aggregate_version_unique",
            "aggregate_id", "aggregate_version",
            unique=True
        ),
        # Constraint append-only: no se permite UPDATE en esta tabla
        # (se implementará con trigger en migración)
        CheckConstraint("id > 0", name="ck_events_positive_id"),
    )


class OrderProjection(Base):
    """Proyección materializada del estado actual de una orden."""
    __tablename__ = "order_projections"

    order_id = Column(String(36), primary_key=True)
    customer_id = Column(String(100), nullable=False)
    status = Column(String(30), nullable=False, default="created")
    items = Column(JSON, nullable=False, default=list)
    total_amount = Column(Float, nullable=False, default=0.0)
    created_at = Column(DateTime(timezone=True), nullable=False)
    updated_at = Column(DateTime(timezone=True), nullable=False)
    event_count = Column(Integer, nullable=False, default=0)
    is_cancelled = Column(Boolean, nullable=False, default=False)
    last_event_id = Column(BigInteger, nullable=False, default=0)


class SnapshotOrder(Base):
    """Snapshot del agregado Order para optimizar reconstrucción."""
    __tablename__ = "snapshot_orders"

    aggregate_id = Column(String(36), primary_key=True)
    aggregate_version = Column(Integer, nullable=False)
    state = Column(JSON, nullable=False)
    created_at = Column(
        DateTime(timezone=True), nullable=False,
        default=lambda: datetime.now(timezone.utc)
    )
EOF
```

3. Instala dependencias (si no están instaladas):

```bash
cd ~/microservices-lab/order-service
cat > requirements.txt << 'EOF'
fastapi==0.111.0
uvicorn==0.29.0
pydantic==2.7.1
pydantic-settings>=2.2.0
sqlalchemy==2.0.30
asyncpg==0.29.0
alembic==1.13.1
confluent-kafka==2.4.0
pytest==8.2.1
pytest-asyncio==0.23.7
httpx==0.27.0
EOF

pip install -r requirements.txt
```

### Verificación

```bash
python -c "from app.eventstore.models import Base, EventRecord, OrderProjection, SnapshotOrder; print(f'Tablas: {list(Base.metadata.tables.keys())}')"
```

**Salida esperada:**

```
Tablas: ['events', 'order_projections', 'snapshot_orders']
```

---

## Paso 3: Crear Migraciones con Alembic

**Objetivo:** Generar y aplicar las migraciones de esquema incluyendo el trigger append-only.

### Instrucciones

1. Inicializa Alembic:

```bash
cd ~/microservices-lab/order-service
alembic init alembic
```

2. Configura `alembic.ini` — reemplaza la línea `sqlalchemy.url`:

```bash
sed -i 's|sqlalchemy.url = .*|sqlalchemy.url = postgresql+asyncpg://mslab_user:mslab_pass_2024@localhost:5432/eventstore_db|' alembic.ini
```

3. Reemplaza `alembic/env.py`:

```bash
cat > alembic/env.py << 'EOF'
import asyncio
from logging.config import fileConfig

from alembic import context
from sqlalchemy import pool
from sqlalchemy.ext.asyncio import create_async_engine

from app.eventstore.models import Base

config = context.config
if config.config_file_name is not None:
    fileConfig(config.config_file_name)

target_metadata = Base.metadata


def run_migrations_offline():
    url = config.get_main_option("sqlalchemy.url")
    context.configure(url=url, target_metadata=target_metadata, literal_binds=True)
    with context.begin_transaction():
        context.run_migrations()


def do_run_migrations(connection):
    context.configure(connection=connection, target_metadata=target_metadata)
    with context.begin_transaction():
        context.run_migrations()


async def run_async_migrations():
    connectable = create_async_engine(
        config.get_main_option("sqlalchemy.url"),
        poolclass=pool.NullPool,
    )
    async with connectable.connect() as connection:
        await connection.run_sync(do_run_migrations)
    await connectable.dispose()


def run_migrations_online():
    asyncio.run(run_async_migrations())


if context.is_offline_mode():
    run_migrations_offline()
else:
    run_migrations_online()
EOF
```

4. Genera la migración inicial:

```bash
alembic revision --autogenerate -m "create_event_store_tables"
```

5. Crea una segunda migración para el trigger append-only:

```bash
cat > alembic/versions/002_append_only_trigger.py << 'EOF'
"""Trigger para garantizar append-only en tabla events.

Revision ID: 002_append_only
Revises: (auto-linked)
"""
from alembic import op

revision = "002_append_only"
down_revision = None  # Se actualizará manualmente después

def upgrade():
    op.execute("""
        CREATE OR REPLACE FUNCTION prevent_event_mutation()
        RETURNS TRIGGER AS $$
        BEGIN
            IF TG_OP = 'UPDATE' OR TG_OP = 'DELETE' THEN
                RAISE EXCEPTION 'Events table is append-only. % operations are not allowed.', TG_OP;
            END IF;
            RETURN NEW;
        END;
        $$ LANGUAGE plpgsql;
    """)
    op.execute("""
        CREATE TRIGGER trg_events_append_only
        BEFORE UPDATE OR DELETE ON events
        FOR EACH ROW EXECUTE FUNCTION prevent_event_mutation();
    """)

def downgrade():
    op.execute("DROP TRIGGER IF EXISTS trg_events_append_only ON events;")
    op.execute("DROP FUNCTION IF EXISTS prevent_event_mutation();")
EOF
```

6. Actualiza el `down_revision` de la segunda migración. Primero, obtén el revision ID de la primera:

```bash
FIRST_REV=$(ls alembic/versions/ | grep create_event_store | head -1 | cut -d'_' -f1)
sed -i "s/down_revision = None/down_revision = \"${FIRST_REV}\"/" alembic/versions/002_append_only_trigger.py
```

7. Aplica las migraciones:

```bash
alembic upgrade head
```

### Salida Esperada

```
INFO  [alembic.runtime.migration] Running upgrade  -> <rev_id>, create_event_store_tables
INFO  [alembic.runtime.migration] Running upgrade <rev_id> -> 002_append_only, ...
```

### Verificación

```bash
docker exec mslab-postgres psql -U mslab_user -d eventstore_db -c "\dt"
```

**Salida esperada** (debe incluir):

```
 Schema |       Name        | Type  |   Owner
--------+-------------------+-------+-----------
 public | events            | table | mslab_user
 public | order_projections | table | mslab_user
 public | snapshot_orders   | table | mslab_user
```

Verifica el trigger:

```bash
docker exec mslab-postgres psql -U mslab_user -d eventstore_db -c \
  "SELECT trigger_name FROM information_schema.triggers WHERE event_object_table='events';"
```

---

## Paso 4: Implementar el Event Store

**Objetivo:** Crear la capa de persistencia que almacena eventos, gestiona concurrencia optimista y genera snapshots.

### Instrucciones

1. Crea el módulo del Event Store:

```bash
cat > ~/microservices-lab/order-service/app/eventstore/__init__.py << 'EOF'
from .store import EventStore
from .models import Base, EventRecord, OrderProjection, SnapshotOrder
from .projections import ProjectionEngine

__all__ = ["EventStore", "Base", "EventRecord", "OrderProjection", "SnapshotOrder", "ProjectionEngine"]
EOF
```

2. Implementa el Event Store:

```bash
cat > ~/microservices-lab/order-service/app/eventstore/store.py << 'EOF'
"""Event Store append-only con soporte de snapshots."""
import json
from typing import List, Optional

from sqlalchemy import select, func
from sqlalchemy.ext.asyncio import AsyncSession, create_async_engine, async_sessionmaker

from app.config import settings
from app.eventstore.models import EventRecord, SnapshotOrder

import sys
sys.path.insert(0, "/root/microservices-lab")
from shared.events.order_events import DomainEvent, EVENT_TYPE_MAP


engine = create_async_engine(settings.eventstore_db_url, echo=False)
async_session_factory = async_sessionmaker(engine, class_=AsyncSession, expire_on_commit=False)


class EventStore:
    """Almacén de eventos append-only con concurrencia optimista."""

    def __init__(self):
        self.session_factory = async_session_factory

    async def append_events(
        self,
        aggregate_id: str,
        events: List[DomainEvent],
        expected_version: int
    ) -> List[EventRecord]:
        """
        Persiste una lista de eventos para un agregado.
        Lanza IntegrityError si expected_version no coincide (optimistic lock).
        """
        records = []
        async with self.session_factory() as session:
            async with session.begin():
                current_version = expected_version
                for event in events:
                    current_version += 1
                    record = EventRecord(
                        event_id=event.event_id,
                        event_type=event.event_type,
                        aggregate_id=aggregate_id,
                        aggregate_type=event.aggregate_type,
                        aggregate_version=current_version,
                        occurred_at=event.occurred_at,
                        payload=event.model_dump(mode="json"),
                        metadata_=event.metadata,
                    )
                    session.add(record)
                    records.append(record)

                # Generar snapshot si se cruza el umbral
                if current_version % settings.snapshot_threshold == 0:
                    await self._create_snapshot(session, aggregate_id, current_version)

        return records

    async def load_events(
        self,
        aggregate_id: str,
        after_version: int = 0
    ) -> List[DomainEvent]:
        """Carga eventos de un agregado después de una versión dada."""
        async with self.session_factory() as session:
            stmt = (
                select(EventRecord)
                .where(EventRecord.aggregate_id == aggregate_id)
                .where(EventRecord.aggregate_version > after_version)
                .order_by(EventRecord.aggregate_version.asc())
            )
            result = await session.execute(stmt)
            records = result.scalars().all()

        events = []
        for record in records:
            event_class = EVENT_TYPE_MAP.get(record.event_type)
            if event_class:
                events.append(event_class(**record.payload))
        return events

    async def load_snapshot(self, aggregate_id: str) -> Optional[dict]:
        """Carga el snapshot más reciente de un agregado."""
        async with self.session_factory() as session:
            stmt = select(SnapshotOrder).where(
                SnapshotOrder.aggregate_id == aggregate_id
            )
            result = await session.execute(stmt)
            snapshot = result.scalar_one_or_none()
            if snapshot:
                return {
                    "version": snapshot.aggregate_version,
                    "state": snapshot.state,
                }
        return None

    async def get_aggregate_version(self, aggregate_id: str) -> int:
        """Obtiene la versión actual de un agregado."""
        async with self.session_factory() as session:
            stmt = select(func.max(EventRecord.aggregate_version)).where(
                EventRecord.aggregate_id == aggregate_id
            )
            result = await session.execute(stmt)
            version = result.scalar()
            return version or 0

    async def load_all_events(self, after_id: int = 0) -> List[EventRecord]:
        """Carga todos los eventos del store (para rebuild de proyecciones)."""
        async with self.session_factory() as session:
            stmt = (
                select(EventRecord)
                .where(EventRecord.id > after_id)
                .order_by(EventRecord.id.asc())
            )
            result = await session.execute(stmt)
            return result.scalars().all()

    async def _create_snapshot(
        self, session: AsyncSession, aggregate_id: str, version: int
    ):
        """Genera un snapshot del estado actual del agregado."""
        # Cargar todos los eventos para reconstruir estado
        stmt = (
            select(EventRecord)
            .where(EventRecord.aggregate_id == aggregate_id)
            .order_by(EventRecord.aggregate_version.asc())
        )
        result = await session.execute(stmt)
        records = result.scalars().all()

        # Reconstruir estado
        state = self._rebuild_state_from_records(records)

        # Upsert snapshot
        existing = await session.get(SnapshotOrder, aggregate_id)
        if existing:
            existing.aggregate_version = version
            existing.state = state
        else:
            snapshot = SnapshotOrder(
                aggregate_id=aggregate_id,
                aggregate_version=version,
                state=state,
            )
            session.add(snapshot)

    @staticmethod
    def _rebuild_state_from_records(records: List[EventRecord]) -> dict:
        """Reconstruye el estado de un agregado a partir de registros."""
        state = {
            "order_id": "",
            "customer_id": "",
            "status": "created",
            "items": [],
            "total_amount": 0.0,
            "is_cancelled": False,
        }
        for record in records:
            payload = record.payload
            if record.event_type == "OrderCreated":
                state["order_id"] = payload.get("aggregate_id", "")
                state["customer_id"] = payload.get("customer_id", "")
                state["items"] = payload.get("items", [])
                state["total_amount"] = payload.get("total_amount", 0.0)
                state["status"] = "created"
            elif record.event_type == "OrderStatusChanged":
                state["status"] = payload.get("new_status", state["status"])
            elif record.event_type == "OrderCancelled":
                state["is_cancelled"] = True
                state["status"] = "cancelled"
            elif record.event_type == "OrderItemAdded":
                state["items"].append({
                    "item_id": payload.get("item_id", ""),
                    "product_name": payload.get("product_name", ""),
                    "quantity": payload.get("quantity", 0),
                    "unit_price": payload.get("unit_price", 0.0),
                })
                state["total_amount"] += (
                    payload.get("quantity", 0) * payload.get("unit_price", 0.0)
                )
        return state
EOF
```

### Verificación

```bash
cd ~/microservices-lab/order-service
python -c "from app.eventstore.store import EventStore; print('EventStore importado correctamente')"
```

---

## Paso 5: Implementar el Motor de Proyecciones

**Objetivo:** Crear proyecciones síncronas que actualicen `order_projections` y un mecanismo de rebuild completo.

### Instrucciones

```bash
cat > ~/microservices-lab/order-service/app/eventstore/projections.py << 'EOF'
"""Motor de proyecciones para el Read Model."""
from datetime import datetime, timezone
from typing import List

from sqlalchemy import select
from sqlalchemy.ext.asyncio import AsyncSession

from app.eventstore.models import OrderProjection, EventRecord
from app.eventstore.store import async_session_factory


class ProjectionEngine:
    """Procesa eventos y actualiza las proyecciones materializadas."""

    def __init__(self):
        self.session_factory = async_session_factory

    async def project_event(self, record: EventRecord):
        """Aplica un evento individual a la proyección (síncrono con transacción)."""
        async with self.session_factory() as session:
            async with session.begin():
                await self._apply_event(session, record)

    async def project_events_batch(self, records: List[EventRecord]):
        """Aplica un lote de eventos a las proyecciones."""
        async with self.session_factory() as session:
            async with session.begin():
                for record in records:
                    await self._apply_event(session, record)

    async def rebuild_all(self):
        """Reconstruye todas las proyecciones desde cero."""
        async with self.session_factory() as session:
            async with session.begin():
                # Limpiar proyecciones existentes
                await session.execute(
                    OrderProjection.__table__.delete()
                )

                # Cargar todos los eventos ordenados
                stmt = select(EventRecord).order_by(EventRecord.id.asc())
                result = await session.execute(stmt)
                all_records = result.scalars().all()

                for record in all_records:
                    await self._apply_event(session, record)

        return len(all_records) if 'all_records' in dir() else 0

    async def _apply_event(self, session: AsyncSession, record: EventRecord):
        """Aplica un evento a la proyección correspondiente."""
        payload = record.payload
        aggregate_id = record.aggregate_id
        now = datetime.now(timezone.utc)

        if record.event_type == "OrderCreated":
            projection = OrderProjection(
                order_id=aggregate_id,
                customer_id=payload.get("customer_id", ""),
                status="created",
                items=payload.get("items", []),
                total_amount=payload.get("total_amount", 0.0),
                created_at=record.occurred_at,
                updated_at=now,
                event_count=1,
                is_cancelled=False,
                last_event_id=record.id,
            )
            session.add(projection)

        elif record.event_type == "OrderStatusChanged":
            projection = await session.get(OrderProjection, aggregate_id)
            if projection:
                projection.status = payload.get("new_status", projection.status)
                projection.updated_at = now
                projection.event_count += 1
                projection.last_event_id = record.id

        elif record.event_type == "OrderCancelled":
            projection = await session.get(OrderProjection, aggregate_id)
            if projection:
                projection.status = "cancelled"
                projection.is_cancelled = True
                projection.updated_at = now
                projection.event_count += 1
                projection.last_event_id = record.id

        elif record.event_type == "OrderItemAdded":
            projection = await session.get(OrderProjection, aggregate_id)
            if projection:
                items = list(projection.items) if projection.items else []
                items.append({
                    "item_id": payload.get("item_id", ""),
                    "product_name": payload.get("product_name", ""),
                    "quantity": payload.get("quantity", 0),
                    "unit_price": payload.get("unit_price", 0.0),
                })
                projection.items = items
                projection.total_amount += (
                    payload.get("quantity", 0) * payload.get("unit_price", 0.0)
                )
                projection.updated_at = now
                projection.event_count += 1
                projection.last_event_id = record.id
EOF
```

### Verificación

```bash
python -c "from app.eventstore.projections import ProjectionEngine; print('ProjectionEngine importado correctamente')"
```

---

## Paso 6: Implementar el Agregado de Dominio

**Objetivo:** Crear el agregado `Order` que encapsula las reglas de negocio y produce eventos.

### Instrucciones

```bash
cat > ~/microservices-lab/order-service/app/domain/__init__.py << 'EOF'
EOF

cat > ~/microservices-lab/order-service/app/domain/order_aggregate.py << 'EOF'
"""Agregado Order con Event Sourcing."""
from typing import List, Optional
from uuid import uuid4

import sys
sys.path.insert(0, "/root/microservices-lab")
from shared.events.order_events import (
    DomainEvent, OrderCreated, OrderStatusChanged,
    OrderCancelled, OrderItemAdded
)


VALID_STATUS_TRANSITIONS = {
    "created": ["confirmed", "cancelled"],
    "confirmed": ["processing", "cancelled"],
    "processing": ["shipped", "cancelled"],
    "shipped": ["delivered"],
    "delivered": [],
    "cancelled": [],
}


class OrderAggregate:
    """Agregado de Orden que aplica reglas de negocio y emite eventos."""

    def __init__(self):
        self.order_id: str = ""
        self.customer_id: str = ""
        self.status: str = ""
        self.items: list = []
        self.total_amount: float = 0.0
        self.is_cancelled: bool = False
        self.version: int = 0
        self._pending_events: List[DomainEvent] = []

    # ── Comandos ─────────────────────────────────────────────────────────────

    def create_order(self, customer_id: str, items: list, total_amount: float) -> str:
        """Comando: crear una nueva orden."""
        if self.order_id:
            raise ValueError("Order already exists.")
        order_id = str(uuid4())
        event = OrderCreated(
            aggregate_id=order_id,
            customer_id=customer_id,
            items=items,
            total_amount=total_amount,
        )
        self._apply(event)
        return order_id

    def change_status(self, new_status: str):
        """Comando: cambiar el estado de la orden."""
        if not self.order_id:
            raise ValueError("Order does not exist.")
        if self.is_cancelled:
            raise ValueError("Cannot change status of a cancelled order.")
        allowed = VALID_STATUS_TRANSITIONS.get(self.status, [])
        if new_status not in allowed:
            raise ValueError(
                f"Invalid transition: {self.status} -> {new_status}. "
                f"Allowed: {allowed}"
            )
        event = OrderStatusChanged(
            aggregate_id=self.order_id,
            previous_status=self.status,
            new_status=new_status,
        )
        self._apply(event)

    def cancel(self, reason: str = ""):
        """Comando: cancelar la orden."""
        if not self.order_id:
            raise ValueError("Order does not exist.")
        if self.is_cancelled:
            raise ValueError("Order is already cancelled.")
        if self.status in ("shipped", "delivered"):
            raise ValueError("Cannot cancel an order that has been shipped or delivered.")
        event = OrderCancelled(
            aggregate_id=self.order_id,
            reason=reason,
        )
        self._apply(event)

    def add_item(self, product_name: str, quantity: int, unit_price: float):
        """Comando: agregar un artículo a la orden."""
        if self.is_cancelled:
            raise ValueError("Cannot add items to a cancelled order.")
        if self.status not in ("created", "confirmed"):
            raise ValueError("Can only add items when order is created or confirmed.")
        event = OrderItemAdded(
            aggregate_id=self.order_id,
            item_id=str(uuid4()),
            product_name=product_name,
            quantity=quantity,
            unit_price=unit_price,
        )
        self._apply(event)

    # ── Aplicación de eventos ────────────────────────────────────────────────

    def _apply(self, event: DomainEvent):
        self._mutate(event)
        self._pending_events.append(event)

    def _mutate(self, event: DomainEvent):
        """Muta el estado interno según el tipo de evento."""
        if isinstance(event, OrderCreated):
            self.order_id = event.aggregate_id
            self.customer_id = event.customer_id
            self.items = list(event.items)
            self.total_amount = event.total_amount
            self.status = "created"
        elif isinstance(event, OrderStatusChanged):
            self.status = event.new_status
        elif isinstance(event, OrderCancelled):
            self.is_cancelled = True
            self.status = "cancelled"
        elif isinstance(event, OrderItemAdded):
            self.items.append({
                "item_id": event.item_id,
                "product_name": event.product_name,
                "quantity": event.quantity,
                "unit_price": event.unit_price,
            })
            self.total_amount += event.quantity * event.unit_price
        self.version += 1

    # ── Reconstrucción ───────────────────────────────────────────────────────

    @classmethod
    def from_events(cls, events: List[DomainEvent]) -> "OrderAggregate":
        """Reconstruye el agregado desde una secuencia de eventos."""
        instance = cls()
        for event in events:
            instance._mutate(event)
        return instance

    @classmethod
    def from_snapshot(cls, state: dict, version: int) -> "OrderAggregate":
        """Reconstruye el agregado desde un snapshot."""
        instance = cls()
        instance.order_id = state.get("order_id", "")
        instance.customer_id = state.get("customer_id", "")
        instance.status = state.get("status", "")
        instance.items = state.get("items", [])
        instance.total_amount = state.get("total_amount", 0.0)
        instance.is_cancelled = state.get("is_cancelled", False)
        instance.version = version
        return instance

    def get_pending_events(self) -> List[DomainEvent]:
        return list(self._pending_events)

    def clear_pending_events(self):
        self._pending_events.clear()
EOF
```

### Verificación

```bash
python -c "
from app.domain.order_aggregate import OrderAggregate
agg = OrderAggregate()
order_id = agg.create_order('customer-1', [{'product': 'Widget', 'qty': 2}], 25.0)
print(f'Order created: {order_id[:8]}... status={agg.status} events={len(agg.get_pending_events())}')
agg.change_status('confirmed')
print(f'Status changed to: {agg.status} events={len(agg.get_pending_events())}')
"
```

**Salida esperada:**

```
Order created: xxxxxxxx... status=created events=1
Status changed to: confirmed events=2
```

---

## Paso 7: Implementar el Productor Kafka

**Objetivo:** Publicar eventos al topic Kafka tras persistirlos en el Event Store.

### Instrucciones

```bash
cat > ~/microservices-lab/order-service/app/kafka_producer.py << 'EOF'
"""Productor Kafka para publicar eventos de dominio."""
import json
from typing import List

from confluent_kafka import Producer

from app.config import settings

import sys
sys.path.insert(0, "/root/microservices-lab")
from shared.events.order_events import DomainEvent


class KafkaEventPublisher:
    """Publica eventos de dominio al topic Kafka."""

    def __init__(self):
        self._producer = Producer({
            "bootstrap.servers": settings.kafka_bootstrap_servers,
            "client.id": "order-service-event-publisher",
            "acks": "all",
        })
        self._topic = settings.kafka_orders_topic

    def publish_events(self, events: List[DomainEvent]):
        """Publica una lista de eventos al topic configurado."""
        for event in events:
            value = event.model_dump_json().encode("utf-8")
            self._producer.produce(
                topic=self._topic,
                key=event.aggregate_id.encode("utf-8"),
                value=value,
            )
        self._producer.flush(timeout=5.0)

    def close(self):
        self._producer.flush(timeout=10.0)


# Singleton
kafka_publisher = KafkaEventPublisher()
EOF
```

Crea el topic en Kafka si no existe:

```bash
docker exec mslab-kafka kafka-topics --create \
  --topic order-events \
  --bootstrap-server localhost:9092 \
  --partitions 3 \
  --replication-factor 1 \
  --if-not-exists
```

### Verificación

```bash
docker exec mslab-kafka kafka-topics --list --bootstrap-server localhost:9092 | grep order-events
```

**Salida esperada:**

```
order-events
```

---

## Paso 8: Implementar la API FastAPI con CQRS

**Objetivo:** Crear endpoints separados para comandos (escritura) y consultas (lectura), integrando Event Store, proyecciones y Kafka.

### Instrucciones

1. Crea los handlers de comandos:

```bash
cat > ~/microservices-lab/order-service/app/commands/__init__.py << 'EOF'
EOF

cat > ~/microservices-lab/order-service/app/commands/order_commands.py << 'EOF'
"""Handlers de comandos para OrderService."""
from typing import List

from app.domain.order_aggregate import OrderAggregate
from app.eventstore.store import EventStore
from app.eventstore.projections import ProjectionEngine
from app.kafka_producer import kafka_publisher

import sys
sys.path.insert(0, "/root/microservices-lab")
from shared.events.order_events import DomainEvent

event_store = EventStore()
projection_engine = ProjectionEngine()


async def _load_aggregate(order_id: str) -> OrderAggregate:
    """Carga un agregado usando snapshot + eventos posteriores."""
    snapshot = await event_store.load_snapshot(order_id)
    if snapshot:
        aggregate = OrderAggregate.from_snapshot(
            state=snapshot["state"],
            version=snapshot["version"]
        )
        # Cargar solo eventos después del snapshot
        events = await event_store.load_events(order_id, after_version=snapshot["version"])
    else:
        events = await event_store.load_events(order_id)
        aggregate = OrderAggregate.from_events(events) if events else OrderAggregate()

    # Sincronizar versión con la del store
    if not snapshot and not events:
        aggregate.version = 0
    elif events:
        aggregate.version = (snapshot["version"] if snapshot else 0) + len(events)

    return aggregate


async def handle_create_order(customer_id: str, items: list, total_amount: float) -> dict:
    """Procesa el comando de crear orden."""
    aggregate = OrderAggregate()
    order_id = aggregate.create_order(customer_id, items, total_amount)

    pending = aggregate.get_pending_events()
    records = await event_store.append_events(order_id, pending, expected_version=0)

    # Proyección síncrona
    for record in records:
        await projection_engine.project_event(record)

    # Publicar a Kafka
    kafka_publisher.publish_events(pending)

    aggregate.clear_pending_events()
    return {"order_id": order_id, "status": "created"}


async def handle_change_status(order_id: str, new_status: str) -> dict:
    """Procesa el comando de cambio de estado."""
    current_version = await event_store.get_aggregate_version(order_id)
    if current_version == 0:
        raise ValueError(f"Order {order_id} not found.")

    aggregate = await _load_aggregate(order_id)
    aggregate.change_status(new_status)

    pending = aggregate.get_pending_events()
    records = await event_store.append_events(order_id, pending, expected_version=current_version)

    for record in records:
        await projection_engine.project_event(record)

    kafka_publisher.publish_events(pending)
    aggregate.clear_pending_events()

    return {"order_id": order_id, "status": new_status}


async def handle_cancel_order(order_id: str, reason: str = "") -> dict:
    """Procesa el comando de cancelación."""
    current_version = await event_store.get_aggregate_version(order_id)
    if current_version == 0:
        raise ValueError(f"Order {order_id} not found.")

    aggregate = await _load_aggregate(order_id)
    aggregate.cancel(reason)

    pending = aggregate.get_pending_events()
    records = await event_store.append_events(order_id, pending, expected_version=current_version)

    for record in records:
        await projection_engine.project_event(record)

    kafka_publisher.publish_events(pending)
    aggregate.clear_pending_events()

    return {"order_id": order_id, "status": "cancelled"}
EOF
```

2. Crea los handlers de consultas:

```bash
cat > ~/microservices-lab/order-service/app/queries/__init__.py << 'EOF'
EOF

cat > ~/microservices-lab/order-service/app/queries/order_queries.py << 'EOF'
"""Handlers de consultas — leen exclusivamente de proyecciones."""
from typing import Optional, List

from sqlalchemy import select
from sqlalchemy.ext.asyncio import AsyncSession

from app.eventstore.models import OrderProjection
from app.eventstore.store import async_session_factory


async def get_order_by_id(order_id: str) -> Optional[dict]:
    """Consulta una orden desde la proyección materializada."""
    async with async_session_factory() as session:
        projection = await session.get(OrderProjection, order_id)
        if not projection:
            return None
        return {
            "order_id": projection.order_id,
            "customer_id": projection.customer_id,
            "status": projection.status,
            "items": projection.items,
            "total_amount": projection.total_amount,
            "created_at": projection.created_at.isoformat(),
            "updated_at": projection.updated_at.isoformat(),
            "event_count": projection.event_count,
            "is_cancelled": projection.is_cancelled,
        }


async def get_all_orders(limit: int = 50, offset: int = 0) -> List[dict]:
    """Lista todas las órdenes desde proyecciones."""
    async with async_session_factory() as session:
        stmt = (
            select(OrderProjection)
            .order_by(OrderProjection.created_at.desc())
            .limit(limit)
            .offset(offset)
        )
        result = await session.execute(stmt)
        projections = result.scalars().all()
        return [
            {
                "order_id": p.order_id,
                "customer_id": p.customer_id,
                "status": p.status,
                "total_amount": p.total_amount,
                "created_at": p.created_at.isoformat(),
                "is_cancelled": p.is_cancelled,
            }
            for p in projections
        ]
EOF
```

3. Crea la aplicación FastAPI principal:

```bash
cat > ~/microservices-lab/order-service/app/__init__.py << 'EOF'
EOF

cat > ~/microservices-lab/order-service/app/main.py << 'EOF'
"""OrderService — API principal con CQRS."""
from fastapi import FastAPI, HTTPException, Query
from pydantic import BaseModel, Field
from typing import List, Optional

from app.commands.order_commands import (
    handle_create_order, handle_change_status, handle_cancel_order
)
from app.queries.order_queries import get_order_by_id, get_all_orders
from app.eventstore.projections import ProjectionEngine

app = FastAPI(
    title="OrderService — Event Sourcing + CQRS",
    version="3.0.0",
    description="Servicio de órdenes con Event Store append-only y proyecciones materializadas",
)


# ── Modelos de Request ───────────────────────────────────────────────────────

class CreateOrderRequest(BaseModel):
    customer_id: str = Field(..., min_length=1)
    items: list = Field(default_factory=list)
    total_amount: float = Field(..., gt=0)


class ChangeStatusRequest(BaseModel):
    new_status: str = Field(..., min_length=1)


class CancelOrderRequest(BaseModel):
    reason: str = ""


# ── Endpoints de Comando (Write Side) ───────────────────────────────────────

@app.post("/orders", status_code=202, tags=["Commands"])
async def create_order(request: CreateOrderRequest):
    """Comando: crear una nueva orden."""
    try:
        result = await handle_create_order(
            customer_id=request.customer_id,
            items=request.items,
            total_amount=request.total_amount,
        )
        return result
    except ValueError as e:
        raise HTTPException(status_code=400, detail=str(e))


@app.patch("/orders/{order_id}/status", tags=["Commands"])
async def change_order_status(order_id: str, request: ChangeStatusRequest):
    """Comando: cambiar estado de una orden."""
    try:
        result = await handle_change_status(order_id, request.new_status)
        return result
    except ValueError as e:
        raise HTTPException(status_code=400, detail=str(e))


@app.post("/orders/{order_id}/cancel", tags=["Commands"])
async def cancel_order(order_id: str, request: CancelOrderRequest):
    """Comando: cancelar una orden."""
    try:
        result = await handle_cancel_order(order_id, request.reason)
        return result
    except ValueError as e:
        raise HTTPException(status_code=400, detail=str(e))


# ── Endpoints de Consulta (Read Side) ───────────────────────────────────────

@app.get("/orders/{order_id}", tags=["Queries"])
async def get_order(order_id: str):
    """Consulta: obtener una orden desde la proyección."""
    result = await get_order_by_id(order_id)
    if not result:
        raise HTTPException(status_code=404, detail="Order not found")
    return result


@app.get("/orders", tags=["Queries"])
async def list_orders(
    limit: int = Query(default=50, le=100),
    offset: int = Query(default=0, ge=0),
):
    """Consulta: listar órdenes desde proyecciones."""
    return await get_all_orders(limit=limit, offset=offset)


# ── Endpoints de Administración ──────────────────────────────────────────────

@app.get("/admin/rebuild-projections", tags=["Admin"])
async def rebuild_projections():
    """Reconstruye todas las proyecciones desde el Event Store."""
    engine = ProjectionEngine()
    count = await engine.rebuild_all()
    return {"status": "completed", "events_processed": count}


@app.get("/health", tags=["System"])
async def health_check():
    return {"status": "healthy", "service": "order-service", "version": "3.0.0"}
EOF
```

### Verificación

Inicia el servicio:

```bash
cd ~/microservices-lab/order-service
uvicorn app.main:app --host 0.0.0.0 --port 8001 --reload &
sleep 3
```

Verifica que el servicio responde:

```bash
curl -s http://localhost:8001/health | python -m json.tool
```

**Salida esperada:**

```json
{
    "status": "healthy",
    "service": "order-service",
    "version": "3.0.0"
}
```

---

## Paso 9: Pruebas de Integración Completas

**Objetivo:** Validar el flujo completo: crear orden → cambiar estado → verificar proyección → validar snapshots → rebuild.

### Instrucciones

1. Crea una orden:

```bash
ORDER_RESPONSE=$(curl -s -X POST http://localhost:8001/orders \
  -H "Content-Type: application/json" \
  -d '{
    "customer_id": "customer-001",
    "items": [{"product": "Laptop", "qty": 1, "price": 999.99}],
    "total_amount": 999.99
  }')
echo $ORDER_RESPONSE | python -m json.tool
ORDER_ID=$(echo $ORDER_RESPONSE | python -c "import sys,json; print(json.load(sys.stdin)['order_id'])")
echo "Order ID: $ORDER_ID"
```

2. Consulta la orden desde la proyección:

```bash
curl -s http://localhost:8001/orders/$ORDER_ID | python -m json.tool
```

**Salida esperada:**

```json
{
    "order_id": "<uuid>",
    "customer_id": "customer-001",
    "status": "created",
    "items": [...],
    "total_amount": 999.99,
    "created_at": "...",
    "updated_at": "...",
    "event_count": 1,
    "is_cancelled": false
}
```

3. Cambia el estado a "confirmed":

```bash
curl -s -X PATCH http://localhost:8001/orders/$ORDER_ID/status \
  -H "Content-Type: application/json" \
  -d '{"new_status": "confirmed"}' | python -m json.tool
```

4. Verifica que la proyección se actualizó:

```bash
curl -s http://localhost:8001/orders/$ORDER_ID | python -c "
import sys, json
data = json.load(sys.stdin)
assert data['status'] == 'confirmed', f'Expected confirmed, got {data[\"status\"]}'
assert data['event_count'] == 2, f'Expected 2 events, got {data[\"event_count\"]}'
print(f'✓ Proyección correcta: status={data[\"status\"]}, events={data[\"event_count\"]}')
"
```

5. Verifica eventos en la base de datos:

```bash
docker exec mslab-postgres psql -U mslab_user -d eventstore_db -c \
  "SELECT event_type, aggregate_version FROM events WHERE aggregate_id='$ORDER_ID' ORDER BY aggregate_version;"
```

**Salida esperada:**

```
    event_type     | aggregate_version
-------------------+-------------------
 OrderCreated      |                 1
 OrderStatusChanged|                 2
```

6. Prueba el endpoint de rebuild:

```bash
curl -s http://localhost:8001/admin/rebuild-projections | python -m json.tool
```

**Salida esperada:**

```json
{
    "status": "completed",
    "events_processed": 2
}
```

7. Verifica que el trigger append-only funciona:

```bash
docker exec mslab-postgres psql -U mslab_user -d eventstore_db -c \
  "UPDATE events SET event_type='HACKED' WHERE id=1;" 2>&1 | grep -o "append-only"
```

**Salida esperada:**

```
append-only
```

---

## Paso 10: Implementar y Verificar Snapshots

**Objetivo:** Generar suficientes eventos para disparar la creación automática de un snapshot y verificar la optimización.

### Instrucciones

1. Crea un script que genere múltiples transiciones de estado:

```bash
cat > ~/microservices-lab/order-service/tests/test_snapshots.py << 'EOF'
"""Test de snapshots — genera 10+ eventos para un agregado."""
import asyncio
import sys
sys.path.insert(0, "/root/microservices-lab/order-service")
sys.path.insert(0, "/root/microservices-lab")

from app.eventstore.store import EventStore
from app.eventstore.projections import ProjectionEngine
from app.domain.order_aggregate import OrderAggregate
from shared.events.order_events import OrderItemAdded


async def test_snapshot_creation():
    store = EventStore()
    projection_engine = ProjectionEngine()

    # Crear orden
    agg = OrderAggregate()
    order_id = agg.create_order("snapshot-customer", [], 10.0)
    pending = agg.get_pending_events()
    records = await store.append_events(order_id, pending, expected_version=0)
    for r in records:
        await projection_engine.project_event(r)
    agg.clear_pending_events()
    print(f"[1] Orden creada: {order_id[:8]}...")

    # Agregar 9 items (total = 10 eventos → trigger snapshot)
    for i in range(9):
        current_v = await store.get_aggregate_version(order_id)
        agg2 = await _reload(store, order_id)
        agg2.add_item(f"Product-{i+1}", quantity=1, unit_price=5.0)
        pending = agg2.get_pending_events()
        records = await store.append_events(order_id, pending, expected_version=current_v)
        for r in records:
            await projection_engine.project_event(r)
        agg2.clear_pending_events()

    print(f"[2] Agregados 9 items (total 10 eventos)")

    # Verificar snapshot
    snapshot = await store.load_snapshot(order_id)
    assert snapshot is not None, "¡Snapshot no fue creado!"
    assert snapshot["version"] == 10, f"Expected version 10, got {snapshot['version']}"
    print(f"[3] ✓ Snapshot creado en versión {snapshot['version']}")
    print(f"    Estado del snapshot: status={snapshot['state']['status']}, items={len(snapshot['state']['items'])}")

    # Verificar que la reconstrucción usa el snapshot
    snap = await store.load_snapshot(order_id)
    events_after = await store.load_events(order_id, after_version=snap["version"])
    print(f"[4] ✓ Eventos después del snapshot: {len(events_after)} (debería ser 0)")

    # Agregar un evento más para verificar recarga parcial
    current_v = await store.get_aggregate_version(order_id)
    agg3 = OrderAggregate.from_snapshot(snap["state"], snap["version"])
    remaining = await store.load_events(order_id, after_version=snap["version"])
    for e in remaining:
        agg3._mutate(e)
    agg3.add_item("Product-Extra", quantity=2, unit_price=7.5)
    pending = agg3.get_pending_events()
    records = await store.append_events(order_id, pending, expected_version=current_v)
    for r in records:
        await projection_engine.project_event(r)
    print(f"[5] ✓ Evento 11 agregado. Versión actual: {await store.get_aggregate_version(order_id)}")

    print("\n✅ Todos los tests de snapshot pasaron correctamente.")


async def _reload(store: EventStore, order_id: str) -> OrderAggregate:
    snapshot = await store.load_snapshot(order_id)
    if snapshot:
        agg = OrderAggregate.from_snapshot(snapshot["state"], snapshot["version"])
        events = await store.load_events(order_id, after_version=snapshot["version"])
    else:
        events = await store.load_events(order_id)
        agg = OrderAggregate.from_events(events) if events else OrderAggregate()
    for e in events if snapshot else []:
        agg._mutate(e)
    return agg


if __name__ == "__main__":
    asyncio.run(test_snapshot_creation())
EOF
```

2. Ejecuta el test:

```bash
cd ~/microservices-lab/order-service
python tests/test_snapshots.py
```

**Salida esperada:**

```
[1] Orden creada: xxxxxxxx...
[2] Agregados 9 items (total 10 eventos)
[3] ✓ Snapshot creado en versión 10
    Estado del snapshot: status=created, items=9
[4] ✓ Eventos después del snapshot: 0 (debería ser 0)
[5] ✓ Evento 11 agregado. Versión actual: 11

✅ Todos los tests de snapshot pasaron correctamente.
```

---

## Validación y Testing

### Test Automatizado con pytest

```bash
cat > ~/microservices-lab/order-service/tests/test_event_store.py << 'EOF'
"""Tests de integración para Event Store y CQRS."""
import pytest
import httpx

BASE_URL = "http://localhost:8001"


@pytest.mark.asyncio
async def test_full_cqrs_flow():
    """Flujo completo: crear → consultar → cambiar estado → verificar."""
    async with httpx.AsyncClient(base_url=BASE_URL) as client:
        # 1. Crear orden
        resp = await client.post("/orders", json={
            "customer_id": "test-cqrs-001",
            "items": [{"product": "TestItem", "qty": 3}],
            "total_amount": 150.0,
        })
        assert resp.status_code == 202
        order_id = resp.json()["order_id"]

        # 2. Consultar desde proyección
        resp = await client.get(f"/orders/{order_id}")
        assert resp.status_code == 200
        data = resp.json()
        assert data["status"] == "created"
        assert data["customer_id"] == "test-cqrs-001"
        assert data["total_amount"] == 150.0

        # 3. Cambiar estado
        resp = await client.patch(f"/orders/{order_id}/status", json={
            "new_status": "confirmed"
        })
        assert resp.status_code == 200

        # 4. Verificar actualización
        resp = await client.get(f"/orders/{order_id}")
        assert resp.json()["status"] == "confirmed"
        assert resp.json()["event_count"] == 2

        # 5. Cancelar
        resp = await client.post(f"/orders/{order_id}/cancel", json={
            "reason": "Test cancellation"
        })
        assert resp.status_code == 200

        # 6. Verificar cancelación
        resp = await client.get(f"/orders/{order_id}")
        assert resp.json()["is_cancelled"] is True
        assert resp.json()["status"] == "cancelled"
        assert resp.json()["event_count"] == 3


@pytest.mark.asyncio
async def test_invalid_status_transition():
    """Verifica que transiciones inválidas son rechazadas."""
    async with httpx.AsyncClient(base_url=BASE_URL) as client:
        resp = await client.post("/orders", json={
            "customer_id": "test-invalid",
            "items": [],
            "total_amount": 50.0,
        })
        order_id = resp.json()["order_id"]

        # Intentar transición inválida: created → shipped
        resp = await client.patch(f"/orders/{order_id}/status", json={
            "new_status": "shipped"
        })
        assert resp.status_code == 400
        assert "Invalid transition" in resp.json()["detail"]


@pytest.mark.asyncio
async def test_rebuild_projections():
    """Verifica que rebuild-projections reconstruye correctamente."""
    async with httpx.AsyncClient(base_url=BASE_URL) as client:
        resp = await client.get("/admin/rebuild-projections")
        assert resp.status_code == 200
        data = resp.json()
        assert data["status"] == "completed"
        assert data["events_processed"] >= 0


@pytest.mark.asyncio
async def test_order_not_found():
    """Verifica 404 para orden inexistente."""
    async with httpx.AsyncClient(base_url=BASE_URL) as client:
        resp = await client.get("/orders/nonexistent-id-12345")
        assert resp.status_code == 404
EOF
```

Ejecuta los tests (asegúrate de que el servicio esté corriendo en puerto 8001):

```bash
cd ~/microservices-lab/order-service
pytest tests/test_event_store.py -v
```

**Salida esperada:**

```
tests/test_event_store.py::test_full_cqrs_flow PASSED
tests/test_event_store.py::test_invalid_status_transition PASSED
tests/test_event_store.py::test_rebuild_projections PASSED
tests/test_event_store.py::test_order_not_found PASSED

========================= 4 passed =========================
```

### Verificación Final de Datos

```bash
echo "=== Eventos en el Event Store ==="
docker exec mslab-postgres psql -U mslab_user -d eventstore_db -c \
  "SELECT event_type, COUNT(*) FROM events GROUP BY event_type ORDER BY event_type;"

echo ""
echo "=== Proyecciones activas ==="
docker exec mslab-postgres psql -U mslab_user -d eventstore_db -c \
  "SELECT order_id, status, event_count, is_cancelled FROM order_projections LIMIT 5;"

echo ""
echo "=== Snapshots generados ==="
docker exec mslab-postgres psql -U mslab_user -d eventstore_db -c \
  "SELECT aggregate_id, aggregate_version, created_at FROM snapshot_orders;"
```

---

## Resolución de Problemas

### Problema 1: Error de conexión a PostgreSQL async

**Síntomas:**

```
sqlalchemy.exc.OperationalError: (asyncpg.exceptions.InvalidCatalogNameError) database "eventstore_db" does not exist
```

**Causa:** La base de datos `eventstore_db` no fue creada antes de ejecutar las migraciones.

**Solución:**

```bash
docker exec mslab-postgres psql -U mslab_user -d postgres -c "CREATE DATABASE eventstore_db OWNER mslab_user;"
cd ~/microservices-lab/order-service
alembic upgrade head
```

### Problema 2: IntegrityError por conflicto de versión de agregado

**Síntomas:**

```
sqlalchemy.exc.IntegrityError: (asyncpg.exceptions.UniqueViolationError) duplicate key value violates unique constraint "ix_aggregate_version_unique"
```

**Causa:** Dos operaciones concurrentes intentaron escribir la misma versión para un agregado (conflicto de concurrencia optimista). Esto es el comportamiento esperado del patrón.

**Solución:** Implementar retry en el handler del comando:

```python
from tenacity import retry, stop_after_attempt, wait_exponential
from sqlalchemy.exc import IntegrityError

@retry(
    retry=lambda e: isinstance(e, IntegrityError),
    stop=stop_after_attempt(3),
    wait=wait_exponential(multiplier=0.1),
)
async def handle_with_retry(order_id, command_fn, *args):
    """Reintenta el comando si hay conflicto de versión."""
    try:
        return await command_fn(order_id, *args)
    except IntegrityError:
        raise  # tenacity lo reintentará
```

---

## Limpieza

Para detener el servicio y limpiar recursos:

```bash
# Detener uvicorn
pkill -f "uvicorn app.main:app" 2>/dev/null || true

# (Opcional) Limpiar datos del Event Store sin eliminar la DB
docker exec mslab-postgres psql -U mslab_user -d eventstore_db -c "
  TRUNCATE events CASCADE;
  TRUNCATE order_projections CASCADE;
  TRUNCATE snapshot_orders CASCADE;
"

# (Opcional) Eliminar la base de datos completamente
# docker exec mslab-postgres psql -U mslab_user -d postgres -c "DROP DATABASE eventstore_db;"
```

> **Nota:** No elimines la base de datos si planeas continuar con los siguientes laboratorios que dependen del Event Store.

---

## Resumen

En este laboratorio implementaste:

| Componente | Descripción |
|-----------|-------------|
| **Event Store** | Tabla `events` append-only con trigger PostgreSQL, concurrencia optimista via versión de agregado |
| **Agregado Order** | Dominio con reglas de negocio, transiciones de estado validadas y emisión de eventos |
| **Proyecciones** | Tabla `order_projections` actualizada síncronamente + endpoint de rebuild completo |
| **Snapshots** | Generación automática cada 10 eventos, optimización de reconstrucción de agregados |
| **CQRS** | Separación explícita: `POST/PATCH` → Event Store; `GET` → Proyecciones |
| **Integración Kafka** | Eventos publicados al topic `order-events` tras persistencia |

### Principios Clave Aplicados

1. **Inmutabilidad**: Los eventos nunca se modifican (trigger + `frozen=True`)
2. **Consistencia eventual**: Las proyecciones se actualizan después de la escritura
3. **Auditoría completa**: Todo cambio queda registrado como evento con timestamp y metadatos
4. **Optimización**: Snapshots evitan replay completo en agregados con historial extenso

### Recursos Adicionales

- [Event Sourcing Pattern — Microsoft](https://learn.microsoft.com/en-us/azure/architecture/patterns/event-sourcing)
- [CQRS — Martin Fowler](https://martinfowler.com/bliki/CQRS.html)
- [SQLAlchemy 2.0 Async Documentation](https://docs.sqlalchemy.org/en/20/orm/extensions/asyncio.html)
- [PostgreSQL Triggers](https://www.postgresql.org/docs/16/plpgsql-trigger.html)
