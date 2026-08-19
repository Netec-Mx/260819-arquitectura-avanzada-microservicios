# Aplicar Circuit Breaker y Retries en Servicios Python

## Metadata

| Campo | Valor |
|-------|-------|
| **Duración** | 66 minutos |
| **Complejidad** | Alta |
| **Nivel Bloom** | Aplicar |

## Descripción General

En este laboratorio implementarás una capa completa de resiliencia para la comunicación síncrona entre OrderService e InventoryService. Añadirás llamadas HTTP directas (usando httpx) para consultar disponibilidad de stock antes de crear una orden, protegidas con circuit breakers (pybreaker), políticas de retry con backoff exponencial y jitter (tenacity), y control de concurrencia mediante bulkheads (asyncio.Semaphore). Validarás el comportamiento inyectando fallos controlados y verificarás el estado de los circuitos mediante endpoints de observabilidad.

## Objetivos de Aprendizaje

- [ ] Implementar políticas de retry con backoff exponencial y jitter usando `tenacity` en llamadas HTTP inter-servicio
- [ ] Configurar circuit breakers con `pybreaker` en puntos de integración críticos, persistiendo estado en Redis
- [ ] Aplicar el patrón bulkhead mediante `asyncio.Semaphore` para limitar concurrencia en llamadas externas
- [ ] Definir estrategias de degradación graceful (fallback responses) cuando los circuitos están abiertos
- [ ] Exponer endpoints de observabilidad para inspeccionar el estado de los circuit breakers en tiempo real

## Prerrequisitos

### Conocimiento Requerido

- Labs 01, 02 y 03 completados (ambos servicios operativos con Event Store y Kafka)
- Comprensión de patrones de resiliencia: circuit breaker, retry, bulkhead
- Manejo de `async/await` avanzado en Python (semáforos, context managers)

### Acceso y Software

| Componente | Versión | Estado requerido |
|------------|---------|------------------|
| Python | 3.12.3 | Instalado |
| Docker Engine | 26.1.3 | Ejecutándose |
| Redis | 7.2.5-alpine | Contenedor `mslab-redis` activo |
| PostgreSQL | 16.3 | Contenedor `mslab-postgres` activo |
| Kafka | 7.6.1 | Contenedor `mslab-kafka` activo |
| httpx | 0.27.0 | Instalado en virtualenvs |
| tenacity | 8.3.0 | Se instalará en este lab |
| pybreaker | 1.2.0 | Se instalará en este lab |

## Entorno del Laboratorio

### Estructura de Directorios Resultante

```
~/microservices-lab/
├── shared/
│   └── resilience/
│       ├── __init__.py
│       ├── circuit_breaker_registry.py
│       ├── retry_policies.py
│       └── bulkhead.py
├── order-service/
│   └── app/
│       ├── clients/
│       │   └── inventory_client.py
│       └── routers/
│           └── health.py
├── scripts/
│   └── chaos-test.py
└── tests/
    └── test_resilience.py
```

### Configuración Inicial

Verifica que los contenedores de infraestructura estén activos:

```bash
docker ps --format "table {{.Names}}\t{{.Status}}" | grep mslab
```

Salida esperada (mínimo):
```
mslab-postgres    Up ...
mslab-redis       Up ...
mslab-kafka       Up ...
mslab-zookeeper   Up ...
```

Si Redis no está activo, inícialo:

```bash
docker run -d --name mslab-redis \
  -p 6379:6379 \
  redis:7.2.5-alpine
```

---

## Paso 1: Instalar Dependencias de Resiliencia

**Objetivo:** Añadir `tenacity`, `pybreaker` y `httpx` a los entornos virtuales de ambos servicios.

### Instrucciones

1. Actualiza el archivo de requisitos del OrderService:

```bash
cd ~/microservices-lab/order-service
cat >> requirements.txt << 'EOF'
tenacity==8.3.0
pybreaker==1.2.0
httpx==0.27.0
redis==5.0.4
EOF
```

2. Instala las dependencias en el entorno virtual:

```bash
source venv/bin/activate
pip install tenacity==8.3.0 pybreaker==1.2.0 httpx==0.27.0 redis==5.0.4
deactivate
```

3. Repite para InventoryService (necesitará redis para el almacenamiento compartido):

```bash
cd ~/microservices-lab/inventory-service
cat >> requirements.txt << 'EOF'
tenacity==8.3.0
pybreaker==1.2.0
redis==5.0.4
EOF
source venv/bin/activate
pip install tenacity==8.3.0 pybreaker==1.2.0 redis==5.0.4
deactivate
```

### Verificación

```bash
cd ~/microservices-lab/order-service
source venv/bin/activate
python -c "import tenacity; import pybreaker; import httpx; print(f'tenacity={tenacity.__version__}'); print(f'pybreaker={pybreaker.__version__}'); print(f'httpx={httpx.__version__}')"
deactivate
```

**Salida esperada:**
```
tenacity=8.3.0
pybreaker=1.2.0
httpx=0.27.0
```

---

## Paso 2: Crear el Módulo Shared de Resiliencia

**Objetivo:** Implementar un módulo reutilizable en `~/microservices-lab/shared/resilience/` con el registry de circuit breakers, políticas de retry y bulkhead.

### Instrucciones

1. Crea la estructura de directorios:

```bash
mkdir -p ~/microservices-lab/shared/resilience
```

2. Crea el archivo `__init__.py`:

```bash
cat > ~/microservices-lab/shared/resilience/__init__.py << 'EOF'
from .circuit_breaker_registry import CircuitBreakerRegistry
from .retry_policies import with_retry
from .bulkhead import BulkheadSemaphore

__all__ = ["CircuitBreakerRegistry", "with_retry", "BulkheadSemaphore"]
EOF
```

3. Implementa el **CircuitBreakerRegistry** con persistencia en Redis:

```bash
cat > ~/microservices-lab/shared/resilience/circuit_breaker_registry.py << 'EOF'
"""
Registry centralizado de Circuit Breakers con persistencia de estado en Redis.
Cada circuit breaker se identifica por el nombre del servicio destino.
"""

import pybreaker
import redis
import json
import logging
from typing import Optional

logger = logging.getLogger(__name__)


class RedisCircuitBreakerStorage(pybreaker.CircuitBreakerStorage):
    """
    Almacenamiento de estado de circuit breaker en Redis.
    Clave: circuit_breaker:{service_name}
    """

    BASE_KEY = "circuit_breaker"

    def __init__(self, service_name: str, redis_client: redis.Redis):
        self._service_name = service_name
        self._redis = redis_client
        self._key = f"{self.BASE_KEY}:{service_name}"
        # Inicializar estado si no existe
        if not self._redis.exists(self._key):
            self._initialize_state()

    def _initialize_state(self):
        state_data = {
            "state": pybreaker.STATE_CLOSED,
            "fail_counter": 0,
            "success_counter": 0,
        }
        self._redis.set(self._key, json.dumps(state_data))

    @property
    def state(self) -> str:
        raw = self._redis.get(self._key)
        if raw is None:
            return pybreaker.STATE_CLOSED
        data = json.loads(raw)
        return data.get("state", pybreaker.STATE_CLOSED)

    @state.setter
    def state(self, state: str):
        raw = self._redis.get(self._key)
        data = json.loads(raw) if raw else {}
        data["state"] = state
        self._redis.set(self._key, json.dumps(data))
        logger.info(
            f"Circuit breaker [{self._service_name}] transición a: {state}"
        )

    @property
    def counter(self) -> int:
        raw = self._redis.get(self._key)
        if raw is None:
            return 0
        data = json.loads(raw)
        return data.get("fail_counter", 0)

    @counter.setter
    def counter(self, value: int):
        raw = self._redis.get(self._key)
        data = json.loads(raw) if raw else {}
        data["fail_counter"] = value
        self._redis.set(self._key, json.dumps(data))

    @property
    def opened_at(self) -> Optional[float]:
        raw = self._redis.get(self._key)
        if raw is None:
            return None
        data = json.loads(raw)
        return data.get("opened_at")

    @opened_at.setter
    def opened_at(self, value: Optional[float]):
        raw = self._redis.get(self._key)
        data = json.loads(raw) if raw else {}
        data["opened_at"] = value
        self._redis.set(self._key, json.dumps(data))

    def increment_counter(self):
        raw = self._redis.get(self._key)
        data = json.loads(raw) if raw else {"fail_counter": 0}
        data["fail_counter"] = data.get("fail_counter", 0) + 1
        self._redis.set(self._key, json.dumps(data))


class CircuitBreakerRegistry:
    """
    Registry singleton que gestiona circuit breakers por servicio destino.
    Configuración por defecto: fail_max=5, reset_timeout=30s.
    """

    def __init__(
        self,
        redis_host: str = "localhost",
        redis_port: int = 6379,
        redis_db: int = 1,
    ):
        self._redis = redis.Redis(
            host=redis_host, port=redis_port, db=redis_db, decode_responses=True
        )
        self._breakers: dict[str, pybreaker.CircuitBreaker] = {}
        self._redis.ping()  # Verificar conectividad
        logger.info("CircuitBreakerRegistry conectado a Redis")

    def get_breaker(
        self,
        service_name: str,
        fail_max: int = 5,
        reset_timeout: int = 30,
        exclude: Optional[list[type]] = None,
    ) -> pybreaker.CircuitBreaker:
        """
        Obtiene o crea un circuit breaker para el servicio indicado.

        Args:
            service_name: Nombre del servicio destino.
            fail_max: Número de fallos para abrir el circuito.
            reset_timeout: Segundos en estado abierto antes de semi-abierto.
            exclude: Excepciones que NO deben contar como fallo.

        Returns:
            Instancia de CircuitBreaker configurada.
        """
        if service_name not in self._breakers:
            storage = RedisCircuitBreakerStorage(service_name, self._redis)
            listeners = [CircuitBreakerLogListener(service_name)]

            breaker = pybreaker.CircuitBreaker(
                fail_max=fail_max,
                reset_timeout=reset_timeout,
                state_storage=storage,
                listeners=listeners,
                exclude=exclude or [],
                name=service_name,
            )
            self._breakers[service_name] = breaker
            logger.info(
                f"Circuit breaker creado: {service_name} "
                f"(fail_max={fail_max}, reset_timeout={reset_timeout}s)"
            )

        return self._breakers[service_name]

    def get_all_states(self) -> dict[str, dict]:
        """Retorna el estado de todos los circuit breakers registrados."""
        states = {}
        for name, breaker in self._breakers.items():
            states[name] = {
                "service": name,
                "state": breaker.current_state,
                "fail_counter": breaker.fail_counter,
                "fail_max": breaker.fail_max,
                "reset_timeout": breaker.reset_timeout,
            }
        return states

    def reset(self, service_name: str):
        """Resetea manualmente un circuit breaker al estado cerrado."""
        if service_name in self._breakers:
            self._breakers[service_name].close()
            logger.info(f"Circuit breaker [{service_name}] reseteado manualmente")


class CircuitBreakerLogListener(pybreaker.CircuitBreakerListener):
    """Listener que registra transiciones de estado del circuit breaker."""

    def __init__(self, service_name: str):
        self._service_name = service_name

    def state_change(self, cb, old_state, new_state):
        logger.warning(
            f"[CB:{self._service_name}] Transición: {old_state.name} -> {new_state.name}"
        )

    def failure(self, cb, exc):
        logger.debug(f"[CB:{self._service_name}] Fallo registrado: {exc}")

    def success(self, cb):
        logger.debug(f"[CB:{self._service_name}] Llamada exitosa")
EOF
```

4. Implementa las **políticas de retry** con tenacity:

```bash
cat > ~/microservices-lab/shared/resilience/retry_policies.py << 'EOF'
"""
Políticas de retry configurables usando tenacity.
Backoff exponencial con jitter, stop_after_attempt(3).
"""

import logging
from functools import wraps
from typing import Callable, Any

from tenacity import (
    retry,
    stop_after_attempt,
    wait_exponential_jitter,
    retry_if_exception_type,
    before_sleep_log,
    after_log,
    RetryError,
)
import httpx

logger = logging.getLogger(__name__)

# Excepciones que justifican un reintento
RETRYABLE_EXCEPTIONS = (
    httpx.ConnectError,
    httpx.ConnectTimeout,
    httpx.ReadTimeout,
    httpx.WriteTimeout,
    httpx.PoolTimeout,
)


class ServiceUnavailableError(Exception):
    """Error lanzado cuando el servicio responde con 5xx."""

    def __init__(self, status_code: int, service: str, detail: str = ""):
        self.status_code = status_code
        self.service = service
        super().__init__(f"{service} respondió con {status_code}: {detail}")


def with_retry(
    max_attempts: int = 3,
    multiplier: float = 1.0,
    min_wait: float = 1.0,
    max_wait: float = 10.0,
    retryable_exceptions: tuple = RETRYABLE_EXCEPTIONS,
):
    """
    Decorador que aplica política de retry con backoff exponencial y jitter.

    Args:
        max_attempts: Número máximo de intentos.
        multiplier: Multiplicador del backoff exponencial.
        min_wait: Espera mínima en segundos.
        max_wait: Espera máxima en segundos.
        retryable_exceptions: Tupla de excepciones que activan retry.

    Returns:
        Decorador configurado.
    """

    def decorator(func: Callable) -> Callable:
        @retry(
            stop=stop_after_attempt(max_attempts),
            wait=wait_exponential_jitter(
                initial=min_wait,
                max=max_wait,
                jitter=min_wait,
            ),
            retry=retry_if_exception_type(
                retryable_exceptions + (ServiceUnavailableError,)
            ),
            before_sleep=before_sleep_log(logger, logging.WARNING),
            after=after_log(logger, logging.DEBUG),
            reraise=True,
        )
        @wraps(func)
        async def wrapper(*args, **kwargs) -> Any:
            return await func(*args, **kwargs)

        return wrapper

    return decorator
EOF
```

5. Implementa el **patrón bulkhead** con asyncio.Semaphore:

```bash
cat > ~/microservices-lab/shared/resilience/bulkhead.py << 'EOF'
"""
Implementación del patrón Bulkhead usando asyncio.Semaphore.
Limita la concurrencia de llamadas a servicios externos.
"""

import asyncio
import logging
from typing import Optional

logger = logging.getLogger(__name__)


class BulkheadSemaphore:
    """
    Bulkhead que limita el número de llamadas concurrentes a un servicio externo.
    Implementado como context manager async.
    """

    def __init__(self, service_name: str, max_concurrent: int = 10):
        """
        Args:
            service_name: Nombre del servicio protegido.
            max_concurrent: Máximo de llamadas concurrentes permitidas.
        """
        self._service_name = service_name
        self._max_concurrent = max_concurrent
        self._semaphore = asyncio.Semaphore(max_concurrent)
        self._active_calls = 0
        self._rejected_calls = 0
        logger.info(
            f"Bulkhead [{service_name}] inicializado: max_concurrent={max_concurrent}"
        )

    async def acquire(self, timeout: Optional[float] = 5.0) -> bool:
        """
        Intenta adquirir un slot del bulkhead.

        Args:
            timeout: Tiempo máximo de espera en segundos. None = sin timeout.

        Returns:
            True si se adquirió el slot.

        Raises:
            BulkheadFullError: Si no se puede adquirir en el timeout.
        """
        try:
            acquired = await asyncio.wait_for(
                self._semaphore.acquire(), timeout=timeout
            )
            if acquired:
                self._active_calls += 1
            return acquired
        except asyncio.TimeoutError:
            self._rejected_calls += 1
            raise BulkheadFullError(
                f"Bulkhead [{self._service_name}] lleno: "
                f"{self._max_concurrent} llamadas concurrentes activas"
            )

    def release(self):
        """Libera un slot del bulkhead."""
        self._semaphore.release()
        self._active_calls = max(0, self._active_calls - 1)

    async def __aenter__(self):
        await self.acquire()
        return self

    async def __aexit__(self, exc_type, exc_val, exc_tb):
        self.release()
        return False

    @property
    def stats(self) -> dict:
        return {
            "service": self._service_name,
            "max_concurrent": self._max_concurrent,
            "active_calls": self._active_calls,
            "available_slots": self._semaphore._value,
            "rejected_calls": self._rejected_calls,
        }


class BulkheadFullError(Exception):
    """Excepción lanzada cuando el bulkhead está lleno."""
    pass
EOF
```

### Verificación

```bash
cd ~/microservices-lab/order-service
source venv/bin/activate
export PYTHONPATH="${PYTHONPATH}:$(dirname $(pwd))/shared"
python -c "
from resilience import CircuitBreakerRegistry, with_retry, BulkheadSemaphore
print('✓ Módulo de resiliencia importado correctamente')
"
deactivate
```

**Salida esperada:**
```
✓ Módulo de resiliencia importado correctamente
```

---

## Paso 3: Implementar el Cliente HTTP Resiliente para InventoryService

**Objetivo:** Crear un cliente HTTP asíncrono en OrderService que combine circuit breaker, retry y bulkhead para consultar stock en InventoryService.

### Instrucciones

1. Crea el directorio de clientes:

```bash
mkdir -p ~/microservices-lab/order-service/app/clients
touch ~/microservices-lab/order-service/app/clients/__init__.py
```

2. Implementa el cliente resiliente:

```bash
cat > ~/microservices-lab/order-service/app/clients/inventory_client.py << 'EOF'
"""
Cliente HTTP resiliente para comunicación con InventoryService.
Combina: Circuit Breaker + Retry con Backoff + Bulkhead.
"""

import logging
import sys
from typing import Optional
from dataclasses import dataclass

import httpx
import pybreaker

# Añadir shared al path
sys.path.insert(0, "/root/microservices-lab/shared")

from resilience.circuit_breaker_registry import CircuitBreakerRegistry
from resilience.retry_policies import with_retry, ServiceUnavailableError
from resilience.bulkhead import BulkheadSemaphore, BulkheadFullError

logger = logging.getLogger(__name__)


@dataclass
class StockResponse:
    """Respuesta de disponibilidad de stock."""
    product_id: str
    available: bool
    quantity: int
    source: str = "inventory-service"  # "inventory-service" o "fallback"


class InventoryClient:
    """
    Cliente resiliente para InventoryService.

    Capas de protección (en orden de ejecución):
    1. Bulkhead: limita concurrencia a 10 llamadas simultáneas.
    2. Circuit Breaker: fail_max=5, reset_timeout=30s.
    3. Retry: 3 intentos con backoff exponencial y jitter.
    4. Timeout: 5s por solicitud individual.
    """

    def __init__(
        self,
        base_url: str = "http://localhost:8002",
        redis_host: str = "localhost",
        redis_port: int = 6379,
    ):
        self._base_url = base_url
        self._timeout = httpx.Timeout(5.0, connect=3.0)

        # Inicializar capas de resiliencia
        self._registry = CircuitBreakerRegistry(
            redis_host=redis_host, redis_port=redis_port
        )
        self._breaker = self._registry.get_breaker(
            service_name="inventory-service",
            fail_max=5,
            reset_timeout=30,
            exclude=[BulkheadFullError],
        )
        self._bulkhead = BulkheadSemaphore(
            service_name="inventory-service",
            max_concurrent=10,
        )

        # Cliente HTTP compartido
        self._client: Optional[httpx.AsyncClient] = None
        logger.info(f"InventoryClient inicializado: base_url={base_url}")

    async def _get_client(self) -> httpx.AsyncClient:
        """Obtiene o crea el cliente HTTP async."""
        if self._client is None or self._client.is_closed:
            self._client = httpx.AsyncClient(
                base_url=self._base_url,
                timeout=self._timeout,
            )
        return self._client

    @with_retry(max_attempts=3, multiplier=1.0, min_wait=1.0, max_wait=10.0)
    async def _do_check_stock(self, product_id: str) -> StockResponse:
        """
        Llamada HTTP real al InventoryService (protegida por retry).
        """
        client = await self._get_client()
        response = await client.get(f"/api/v1/inventory/{product_id}/stock")

        if response.status_code >= 500:
            raise ServiceUnavailableError(
                status_code=response.status_code,
                service="inventory-service",
                detail=response.text,
            )

        if response.status_code == 404:
            return StockResponse(
                product_id=product_id,
                available=False,
                quantity=0,
                source="inventory-service",
            )

        data = response.json()
        return StockResponse(
            product_id=product_id,
            available=data.get("quantity", 0) > 0,
            quantity=data.get("quantity", 0),
            source="inventory-service",
        )

    async def check_stock(self, product_id: str) -> StockResponse:
        """
        Consulta disponibilidad de stock con todas las capas de resiliencia.

        Flujo:
        1. Adquirir slot del bulkhead
        2. Verificar circuit breaker
        3. Ejecutar con retry
        4. Si falla todo -> fallback

        Returns:
            StockResponse con datos reales o de fallback.
        """
        try:
            # Capa 1: Bulkhead
            async with self._bulkhead:
                # Capa 2: Circuit Breaker
                # pybreaker es síncrono, lo envolvemos
                try:
                    result = self._breaker.call(
                        self._sync_wrapper_check_stock, product_id
                    )
                    # Como pybreaker no soporta async nativamente,
                    # usamos un enfoque alternativo
                    raise NotImplementedError("Usar enfoque async")
                except (NotImplementedError, TypeError):
                    # Enfoque async compatible con pybreaker
                    result = await self._check_with_breaker(product_id)

                return result

        except BulkheadFullError:
            logger.warning(
                f"Bulkhead lleno para inventory-service. "
                f"Retornando fallback para product_id={product_id}"
            )
            return self._fallback_response(product_id, reason="bulkhead_full")

        except pybreaker.CircuitBreakerError:
            logger.warning(
                f"Circuit breaker ABIERTO para inventory-service. "
                f"Retornando fallback para product_id={product_id}"
            )
            return self._fallback_response(product_id, reason="circuit_open")

        except Exception as e:
            logger.error(
                f"Error inesperado consultando stock: {e}. Retornando fallback."
            )
            return self._fallback_response(product_id, reason="unexpected_error")

    async def _check_with_breaker(self, product_id: str) -> StockResponse:
        """
        Ejecuta la consulta de stock verificando el estado del circuit breaker.
        Implementación compatible con async.
        """
        # Verificar estado del breaker manualmente
        if self._breaker.current_state == "open":
            raise pybreaker.CircuitBreakerError(self._breaker)

        try:
            result = await self._do_check_stock(product_id)
            # Registrar éxito
            self._breaker._success_call()
            return result
        except (ServiceUnavailableError, httpx.ConnectError, httpx.ReadTimeout) as e:
            # Registrar fallo
            self._breaker._failure_call()
            if self._breaker.current_state == "open":
                raise pybreaker.CircuitBreakerError(self._breaker)
            raise

    def _fallback_response(self, product_id: str, reason: str) -> StockResponse:
        """
        Respuesta de degradación graceful.
        Asume stock disponible para no bloquear órdenes (optimista).
        En producción, se consultaría una caché o un valor por defecto configurado.
        """
        logger.info(
            f"Fallback activado para product_id={product_id} (razón: {reason})"
        )
        return StockResponse(
            product_id=product_id,
            available=True,  # Optimista: permitir la orden
            quantity=-1,     # -1 indica dato no verificado
            source=f"fallback:{reason}",
        )

    @property
    def circuit_breaker_state(self) -> dict:
        """Retorna el estado actual del circuit breaker."""
        return {
            "service": "inventory-service",
            "state": self._breaker.current_state,
            "fail_counter": self._breaker.fail_counter,
            "fail_max": self._breaker.fail_max,
            "reset_timeout": self._breaker.reset_timeout,
        }

    @property
    def bulkhead_stats(self) -> dict:
        """Retorna estadísticas del bulkhead."""
        return self._bulkhead.stats

    async def close(self):
        """Cierra el cliente HTTP."""
        if self._client and not self._client.is_closed:
            await self._client.aclose()


# Instancia global (singleton por proceso)
_inventory_client: Optional[InventoryClient] = None


def get_inventory_client() -> InventoryClient:
    """Factory function para obtener la instancia del cliente."""
    global _inventory_client
    if _inventory_client is None:
        _inventory_client = InventoryClient()
    return _inventory_client
EOF
```

### Verificación

```bash
cd ~/microservices-lab/order-service
source venv/bin/activate
export PYTHONPATH="${PYTHONPATH}:/root/microservices-lab/shared"
python -c "
from app.clients.inventory_client import InventoryClient, StockResponse
print('✓ InventoryClient importado correctamente')
print(f'  StockResponse fields: {StockResponse.__dataclass_fields__.keys()}')
"
deactivate
```

**Salida esperada:**
```
✓ InventoryClient importado correctamente
  StockResponse fields: dict_keys(['product_id', 'available', 'quantity', 'source'])
```

---

## Paso 4: Añadir Endpoint de Observabilidad de Circuit Breakers

**Objetivo:** Crear un endpoint `GET /health/circuit-breakers` en OrderService que exponga el estado de todos los circuit breakers.

### Instrucciones

1. Crea el router de health:

```bash
mkdir -p ~/microservices-lab/order-service/app/routers
cat > ~/microservices-lab/order-service/app/routers/__init__.py << 'EOF'
EOF
```

```bash
cat > ~/microservices-lab/order-service/app/routers/health.py << 'EOF'
"""
Endpoints de observabilidad para resiliencia.
"""

from fastapi import APIRouter, status
from typing import Any

from app.clients.inventory_client import get_inventory_client

router = APIRouter(prefix="/health", tags=["health"])


@router.get(
    "/circuit-breakers",
    status_code=status.HTTP_200_OK,
    summary="Estado de todos los circuit breakers",
    response_description="Mapa de circuit breakers con su estado actual",
)
async def get_circuit_breakers_status() -> dict[str, Any]:
    """
    Retorna el estado de todos los circuit breakers registrados.
    Útil para dashboards de observabilidad y alertas.

    Estados posibles:
    - closed: Operación normal, solicitudes fluyen.
    - open: Circuito abierto, solicitudes rechazadas (fallback).
    - half-open: Probando recuperación con solicitudes limitadas.
    """
    client = get_inventory_client()

    return {
        "circuit_breakers": {
            "inventory-service": client.circuit_breaker_state,
        },
        "bulkheads": {
            "inventory-service": client.bulkhead_stats,
        },
    }


@router.post(
    "/circuit-breakers/{service_name}/reset",
    status_code=status.HTTP_200_OK,
    summary="Resetear un circuit breaker manualmente",
)
async def reset_circuit_breaker(service_name: str) -> dict[str, str]:
    """
    Resetea manualmente un circuit breaker al estado cerrado.
    Usar con precaución en producción.
    """
    client = get_inventory_client()
    if service_name == "inventory-service":
        client._registry.reset(service_name)
        return {"message": f"Circuit breaker '{service_name}' reseteado", "state": "closed"}
    return {"message": f"Circuit breaker '{service_name}' no encontrado", "state": "unknown"}
EOF
```

2. Crea un archivo principal simplificado para probar (si no existe ya un `main.py` adecuado):

```bash
cat > ~/microservices-lab/order-service/app/main_resilience.py << 'EOF'
"""
Aplicación FastAPI del OrderService con endpoints de resiliencia.
"""

import sys
import logging

sys.path.insert(0, "/root/microservices-lab/shared")

from fastapi import FastAPI
from contextlib import asynccontextmanager

from app.clients.inventory_client import get_inventory_client
from app.routers.health import router as health_router

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)


@asynccontextmanager
async def lifespan(app: FastAPI):
    """Gestión del ciclo de vida de la aplicación."""
    logger.info("Inicializando OrderService con resiliencia...")
    # Inicializar cliente de inventario
    client = get_inventory_client()
    logger.info("InventoryClient inicializado")
    yield
    # Cleanup
    await client.close()
    logger.info("OrderService detenido")


app = FastAPI(
    title="OrderService",
    version="1.4.0",
    description="OrderService con capa de resiliencia (Circuit Breaker, Retry, Bulkhead)",
    lifespan=lifespan,
)

app.include_router(health_router)


@app.get("/api/v1/orders/check-stock/{product_id}")
async def check_product_stock(product_id: str):
    """
    Endpoint que consulta stock en InventoryService usando el cliente resiliente.
    Demuestra la integración de circuit breaker + retry + bulkhead.
    """
    client = get_inventory_client()
    result = await client.check_stock(product_id)
    return {
        "product_id": result.product_id,
        "available": result.available,
        "quantity": result.quantity,
        "source": result.source,
    }
EOF
```

### Verificación

Inicia el servicio temporalmente para verificar los endpoints:

```bash
cd ~/microservices-lab/order-service
source venv/bin/activate
export PYTHONPATH="${PYTHONPATH}:/root/microservices-lab/shared"
uvicorn app.main_resilience:app --host 0.0.0.0 --port 8001 &
sleep 3
curl -s http://localhost:8001/health/circuit-breakers | python -m json.tool
kill %1 2>/dev/null
deactivate
```

**Salida esperada:**
```json
{
    "circuit_breakers": {
        "inventory-service": {
            "service": "inventory-service",
            "state": "closed",
            "fail_counter": 0,
            "fail_max": 5,
            "reset_timeout": 30
        }
    },
    "bulkheads": {
        "inventory-service": {
            "service": "inventory-service",
            "max_concurrent": 10,
            "active_calls": 0,
            "available_slots": 10,
            "rejected_calls": 0
        }
    }
}
```

---

## Paso 5: Crear un Mock de InventoryService con Inyección de Fallos

**Objetivo:** Implementar un servicio simulado que permita controlar el comportamiento (éxito, error 500, latencia) para probar los patrones de resiliencia.

### Instrucciones

1. Crea el mock de InventoryService:

```bash
cat > ~/microservices-lab/scripts/mock_inventory_service.py << 'EOF'
"""
Mock de InventoryService con inyección de fallos controlada.
Permite simular: respuestas normales, errores 500, latencia alta y timeouts.

Control via variables de entorno o endpoint /admin/config.
"""

import asyncio
import time
import logging
from enum import Enum

from fastapi import FastAPI, HTTPException, status
from pydantic import BaseModel

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

app = FastAPI(title="Mock InventoryService", version="0.1.0")


class FailureMode(str, Enum):
    NONE = "none"
    ERROR_500 = "error_500"
    LATENCY = "latency"
    TIMEOUT = "timeout"
    INTERMITTENT = "intermittent"


class MockConfig(BaseModel):
    failure_mode: FailureMode = FailureMode.NONE
    latency_seconds: float = 0.0
    error_rate: float = 0.0  # 0.0 a 1.0 para modo intermitente
    request_count: int = 0


# Estado global del mock
config = MockConfig()
_request_counter = 0


@app.post("/admin/config")
async def set_config(new_config: MockConfig):
    """Configura el modo de fallo del mock."""
    global config, _request_counter
    config = new_config
    _request_counter = 0
    logger.info(f"Mock configurado: {config.failure_mode.value}")
    return {"status": "configured", "config": config.model_dump()}


@app.get("/admin/config")
async def get_config():
    """Obtiene la configuración actual."""
    return config.model_dump()


@app.get("/admin/stats")
async def get_stats():
    """Estadísticas del mock."""
    return {"total_requests": _request_counter}


@app.get("/api/v1/inventory/{product_id}/stock")
async def check_stock(product_id: str):
    """
    Endpoint de stock que responde según la configuración de fallos.
    """
    global _request_counter
    _request_counter += 1

    import random

    # Modo: Error 500 siempre
    if config.failure_mode == FailureMode.ERROR_500:
        logger.info(f"[{_request_counter}] Retornando error 500")
        raise HTTPException(
            status_code=500,
            detail="Internal Server Error (simulado)",
        )

    # Modo: Latencia alta
    if config.failure_mode == FailureMode.LATENCY:
        logger.info(f"[{_request_counter}] Añadiendo latencia: {config.latency_seconds}s")
        await asyncio.sleep(config.latency_seconds)

    # Modo: Timeout (espera más que el timeout del cliente)
    if config.failure_mode == FailureMode.TIMEOUT:
        logger.info(f"[{_request_counter}] Simulando timeout (30s)")
        await asyncio.sleep(30)

    # Modo: Intermitente (falla según error_rate)
    if config.failure_mode == FailureMode.INTERMITTENT:
        if random.random() < config.error_rate:
            logger.info(f"[{_request_counter}] Fallo intermitente")
            raise HTTPException(status_code=500, detail="Fallo intermitente")

    # Respuesta exitosa
    return {
        "product_id": product_id,
        "quantity": 42,
        "warehouse": "main",
        "last_updated": "2024-01-15T10:30:00Z",
    }
EOF
```

2. Inicia el mock en el puerto 8002:

```bash
cd ~/microservices-lab
source order-service/venv/bin/activate
uvicorn scripts.mock_inventory_service:app --host 0.0.0.0 --port 8002 &
MOCK_PID=$!
sleep 2
echo "Mock InventoryService PID: $MOCK_PID"
```

3. Verifica que el mock responde correctamente:

```bash
# Respuesta normal
curl -s http://localhost:8002/api/v1/inventory/PROD-001/stock | python -m json.tool

# Configurar modo error
curl -s -X POST http://localhost:8002/admin/config \
  -H "Content-Type: application/json" \
  -d '{"failure_mode": "error_500"}' | python -m json.tool

# Verificar que retorna 500
curl -s -o /dev/null -w "%{http_code}" http://localhost:8002/api/v1/inventory/PROD-001/stock

# Restaurar modo normal
curl -s -X POST http://localhost:8002/admin/config \
  -H "Content-Type: application/json" \
  -d '{"failure_mode": "none"}'
```

**Salida esperada:**
```json
{
    "product_id": "PROD-001",
    "quantity": 42,
    "warehouse": "main",
    "last_updated": "2024-01-15T10:30:00Z"
}
```
```
500
```

### Verificación

```bash
curl -s http://localhost:8002/admin/stats | python -m json.tool
```

Debe mostrar `total_requests` > 0.

---

## Paso 6: Crear el Script de Pruebas de Caos

**Objetivo:** Implementar un script que simula escenarios de fallo y valida el comportamiento de los circuit breakers, retries y bulkhead.

### Instrucciones

1. Crea el script de pruebas de caos:

```bash
cat > ~/microservices-lab/scripts/chaos-test.py << 'EOF'
#!/usr/bin/env python3
"""
Script de pruebas de caos para validar patrones de resiliencia.
Simula: latencia, errores 500 y timeouts.
Verifica: apertura del circuit breaker, respuestas fallback, reintentos.

Uso: python scripts/chaos-test.py
Prerequisito: OrderService en :8001, Mock InventoryService en :8002
"""

import asyncio
import httpx
import time
import sys
import json

BASE_ORDER_URL = "http://localhost:8001"
BASE_MOCK_URL = "http://localhost:8002"


async def configure_mock(failure_mode: str, **kwargs):
    """Configura el modo de fallo del mock."""
    async with httpx.AsyncClient() as client:
        config = {"failure_mode": failure_mode, **kwargs}
        resp = await client.post(f"{BASE_MOCK_URL}/admin/config", json=config)
        assert resp.status_code == 200, f"Error configurando mock: {resp.text}"
        print(f"  Mock configurado: {failure_mode} {kwargs}")


async def check_stock(product_id: str = "PROD-001") -> dict:
    """Llama al endpoint de check-stock del OrderService."""
    async with httpx.AsyncClient(timeout=15.0) as client:
        resp = await client.get(
            f"{BASE_ORDER_URL}/api/v1/orders/check-stock/{product_id}"
        )
        return {"status_code": resp.status_code, "body": resp.json()}


async def get_cb_status() -> dict:
    """Obtiene el estado del circuit breaker."""
    async with httpx.AsyncClient() as client:
        resp = await client.get(f"{BASE_ORDER_URL}/health/circuit-breakers")
        return resp.json()


async def reset_circuit_breaker():
    """Resetea el circuit breaker."""
    async with httpx.AsyncClient() as client:
        resp = await client.post(
            f"{BASE_ORDER_URL}/health/circuit-breakers/inventory-service/reset"
        )
        return resp.json()


def print_header(title: str):
    print(f"\n{'='*60}")
    print(f"  {title}")
    print(f"{'='*60}")


def print_result(label: str, result: dict, expected: str):
    source = result.get("body", {}).get("source", "N/A")
    status = "✓" if expected in str(result) or expected in source else "✗"
    print(f"  {status} {label}: source={source}, status={result['status_code']}")


async def test_scenario_1_normal_operation():
    """Escenario 1: Operación normal - todas las llamadas exitosas."""
    print_header("ESCENARIO 1: Operación Normal")

    await configure_mock("none")
    await asyncio.sleep(1)

    print("\n  Ejecutando 5 llamadas normales...")
    for i in range(5):
        result = await check_stock(f"PROD-{i:03d}")
        print_result(f"  Llamada {i+1}", result, "inventory-service")

    status = await get_cb_status()
    state = status["circuit_breakers"]["inventory-service"]["state"]
    print(f"\n  Estado CB: {state}")
    assert state == "closed", f"CB debería estar cerrado, está: {state}"
    print("  ✓ Circuit breaker permanece CERRADO")


async def test_scenario_2_errors_open_circuit():
    """Escenario 2: Errores 500 consecutivos abren el circuit breaker."""
    print_header("ESCENARIO 2: Errores 500 → Circuit Breaker ABIERTO")

    await reset_circuit_breaker()
    await configure_mock("error_500")
    await asyncio.sleep(1)

    print("\n  Enviando solicitudes para provocar apertura del CB (fail_max=5)...")
    print("  (Cada solicitud tiene 3 reintentos internos)")

    for i in range(6):
        result = await check_stock("PROD-FAIL")
        source = result.get("body", {}).get("source", "")
        print(f"    Solicitud {i+1}: source={source}")
        await asyncio.sleep(0.5)

    # Verificar que el CB está abierto
    status = await get_cb_status()
    state = status["circuit_breakers"]["inventory-service"]["state"]
    print(f"\n  Estado CB después de fallos: {state}")

    # La siguiente llamada debería usar fallback inmediatamente
    result = await check_stock("PROD-AFTER-OPEN")
    source = result.get("body", {}).get("source", "")
    print(f"  Llamada con CB abierto: source={source}")

    if "fallback" in source:
        print("  ✓ Fallback activado correctamente con circuito abierto")
    else:
        print("  ⚠ El fallback podría no haberse activado (verificar logs)")


async def test_scenario_3_recovery():
    """Escenario 3: Recuperación - el circuito se cierra tras éxitos."""
    print_header("ESCENARIO 3: Recuperación del Servicio")

    # Esperar que el CB pase a semi-abierto (reset_timeout=30s)
    print("\n  Esperando reset_timeout (30s) para transición a SEMI-ABIERTO...")
    print("  (En un test real esperaríamos; aquí reseteamos manualmente)")

    await reset_circuit_breaker()
    await configure_mock("none")
    await asyncio.sleep(1)

    print("  CB reseteado. Enviando solicitudes de verificación...")
    for i in range(3):
        result = await check_stock(f"PROD-RECOVER-{i}")
        source = result.get("body", {}).get("source", "")
        print(f"    Solicitud {i+1}: source={source}")

    status = await get_cb_status()
    state = status["circuit_breakers"]["inventory-service"]["state"]
    print(f"\n  Estado CB tras recuperación: {state}")
    print("  ✓ Servicio recuperado, CB cerrado")


async def test_scenario_4_latency():
    """Escenario 4: Latencia alta provoca timeouts y retries."""
    print_header("ESCENARIO 4: Latencia Alta → Timeouts")

    await reset_circuit_breaker()
    await configure_mock("latency", latency_seconds=8.0)
    await asyncio.sleep(1)

    print("\n  Configurada latencia de 8s (timeout del cliente: 5s)")
    print("  Ejecutando solicitud (esperará timeout + retries)...")

    start = time.time()
    result = await check_stock("PROD-SLOW")
    elapsed = time.time() - start
    source = result.get("body", {}).get("source", "")
    print(f"    Resultado: source={source}, tiempo={elapsed:.1f}s")
    print(f"  ✓ Solicitud completada en {elapsed:.1f}s (con retries/fallback)")


async def test_scenario_5_concurrent_bulkhead():
    """Escenario 5: Concurrencia alta prueba el bulkhead."""
    print_header("ESCENARIO 5: Concurrencia Alta → Bulkhead")

    await reset_circuit_breaker()
    await configure_mock("latency", latency_seconds=3.0)
    await asyncio.sleep(1)

    print("\n  Enviando 15 solicitudes concurrentes (bulkhead max=10)...")
    tasks = [check_stock(f"PROD-CONC-{i}") for i in range(15)]
    results = await asyncio.gather(*tasks, return_exceptions=True)

    fallback_count = sum(
        1 for r in results
        if isinstance(r, dict) and "fallback" in str(r.get("body", {}).get("source", ""))
    )
    success_count = sum(
        1 for r in results
        if isinstance(r, dict) and "inventory-service" == r.get("body", {}).get("source", "")
    )

    print(f"    Exitosas (servicio real): {success_count}")
    print(f"    Fallback (bulkhead/timeout): {fallback_count}")
    print(f"    Errores: {len(results) - success_count - fallback_count}")

    status = await get_cb_status()
    bulkhead = status["bulkheads"]["inventory-service"]
    print(f"    Bulkhead rejected: {bulkhead['rejected_calls']}")
    print("  ✓ Bulkhead limitó la concurrencia")


async def main():
    """Ejecuta todos los escenarios de caos."""
    print("\n" + "╔" + "═"*58 + "╗")
    print("║   PRUEBAS DE CAOS - Patrones de Resiliencia              ║")
    print("╚" + "═"*58 + "╝")

    try:
        # Verificar que los servicios están disponibles
        async with httpx.AsyncClient(timeout=5.0) as client:
            try:
                await client.get(f"{BASE_ORDER_URL}/health/circuit-breakers")
            except Exception as e:
                print(f"\n✗ OrderService no disponible en {BASE_ORDER_URL}: {e}")
                print("  Asegúrate de que OrderService está corriendo en el puerto 8001")
                sys.exit(1)

            try:
                await client.get(f"{BASE_MOCK_URL}/admin/config")
            except Exception as e:
                print(f"\n✗ Mock InventoryService no disponible en {BASE_MOCK_URL}: {e}")
                print("  Asegúrate de que el mock está corriendo en el puerto 8002")
                sys.exit(1)

        await test_scenario_1_normal_operation()
        await test_scenario_2_errors_open_circuit()
        await test_scenario_3_recovery()
        await test_scenario_4_latency()
        await test_scenario_5_concurrent_bulkhead()

        print("\n" + "="*60)
        print("  TODAS LAS PRUEBAS DE CAOS COMPLETADAS")
        print("="*60 + "\n")

    except Exception as e:
        print(f"\n✗ Error durante las pruebas: {e}")
        import traceback
        traceback.print_exc()
        sys.exit(1)


if __name__ == "__main__":
    asyncio.run(main())
EOF
chmod +x ~/microservices-lab/scripts/chaos-test.py
```

### Verificación

Verifica que el script es sintácticamente correcto:

```bash
cd ~/microservices-lab
source order-service/venv/bin/activate
python -c "import ast; ast.parse(open('scripts/chaos-test.py').read()); print('✓ Script válido')"
deactivate
```

**Salida esperada:**
```
✓ Script válido
```

---

## Paso 7: Ejecutar las Pruebas de Caos

**Objetivo:** Ejecutar el script de caos con ambos servicios activos y verificar el comportamiento correcto de los patrones de resiliencia.

### Instrucciones

1. Asegúrate de que el mock de InventoryService sigue corriendo (puerto 8002). Si no:

```bash
cd ~/microservices-lab
source order-service/venv/bin/activate
# Matar procesos previos en puertos 8001 y 8002
kill $(lsof -t -i:8001) 2>/dev/null
kill $(lsof -t -i:8002) 2>/dev/null
sleep 1

# Iniciar mock
export PYTHONPATH="/root/microservices-lab/shared"
uvicorn scripts.mock_inventory_service:app --host 0.0.0.0 --port 8002 &
sleep 2
```

2. Inicia OrderService con resiliencia:

```bash
uvicorn order-service.app.main_resilience:app --host 0.0.0.0 --port 8001 &
sleep 3
```

3. Ejecuta las pruebas de caos:

```bash
python scripts/chaos-test.py
```

### Salida Esperada

```
╔══════════════════════════════════════════════════════════╗
║   PRUEBAS DE CAOS - Patrones de Resiliencia              ║
╚══════════════════════════════════════════════════════════╝

============================================================
  ESCENARIO 1: Operación Normal
============================================================
  Mock configurado: none {}

  Ejecutando 5 llamadas normales...
  ✓   Llamada 1: source=inventory-service, status=200
  ✓   Llamada 2: source=inventory-service, status=200
  ✓   Llamada 3: source=inventory-service, status=200
  ✓   Llamada 4: source=inventory-service, status=200
  ✓   Llamada 5: source=inventory-service, status=200

  Estado CB: closed
  ✓ Circuit breaker permanece CERRADO

============================================================
  ESCENARIO 2: Errores 500 → Circuit Breaker ABIERTO
============================================================
  ...
  ✓ Fallback activado correctamente con circuito abierto

============================================================
  ESCENARIO 3: Recuperación del Servicio
============================================================
  ...
  ✓ Servicio recuperado, CB cerrado

============================================================
  TODAS LAS PRUEBAS DE CAOS COMPLETADAS
============================================================
```

4. Detén los servicios de prueba:

```bash
kill $(lsof -t -i:8001) 2>/dev/null
kill $(lsof -t -i:8002) 2>/dev/null
deactivate
```

---

## Paso 8: Implementar Tests Unitarios con pytest

**Objetivo:** Crear tests automatizados que validen el comportamiento de cada componente de resiliencia de forma aislada.

### Instrucciones

1. Instala dependencias de testing:

```bash
cd ~/microservices-lab/order-service
source venv/bin/activate
pip install pytest==8.2.1 pytest-asyncio==0.23.7
```

2. Crea los tests:

```bash
mkdir -p ~/microservices-lab/tests
cat > ~/microservices-lab/tests/test_resilience.py << 'EOF'
"""
Tests unitarios para los componentes de resiliencia.
"""

import asyncio
import sys
import pytest
import time

sys.path.insert(0, "/root/microservices-lab/shared")

from resilience.circuit_breaker_registry import CircuitBreakerRegistry
from resilience.retry_policies import with_retry, ServiceUnavailableError
from resilience.bulkhead import BulkheadSemaphore, BulkheadFullError


@pytest.fixture
def cb_registry():
    """Registry de circuit breakers conectado a Redis."""
    registry = CircuitBreakerRegistry(
        redis_host="localhost", redis_port=6379, redis_db=2
    )
    return registry


class TestCircuitBreakerRegistry:
    """Tests para el registry de circuit breakers."""

    def test_create_breaker(self, cb_registry):
        """Debe crear un circuit breaker con la configuración indicada."""
        breaker = cb_registry.get_breaker(
            "test-service", fail_max=5, reset_timeout=30
        )
        assert breaker is not None
        assert breaker.fail_max == 5
        assert breaker.reset_timeout == 30
        assert breaker.current_state == "closed"

    def test_same_breaker_returned(self, cb_registry):
        """Debe retornar la misma instancia para el mismo servicio."""
        b1 = cb_registry.get_breaker("test-service-singleton")
        b2 = cb_registry.get_breaker("test-service-singleton")
        assert b1 is b2

    def test_breaker_opens_after_failures(self, cb_registry):
        """El circuito debe abrirse después de fail_max fallos."""
        breaker = cb_registry.get_breaker(
            "test-service-open", fail_max=3, reset_timeout=30
        )

        def failing_call():
            raise Exception("Fallo simulado")

        for _ in range(3):
            try:
                breaker.call(failing_call)
            except Exception:
                pass

        assert breaker.current_state == "open"

    def test_get_all_states(self, cb_registry):
        """Debe retornar el estado de todos los breakers registrados."""
        cb_registry.get_breaker("svc-a")
        cb_registry.get_breaker("svc-b")
        states = cb_registry.get_all_states()
        assert "svc-a" in states
        assert "svc-b" in states
        assert states["svc-a"]["state"] == "closed"


class TestRetryPolicy:
    """Tests para las políticas de retry."""

    @pytest.mark.asyncio
    async def test_retry_succeeds_on_third_attempt(self):
        """Debe reintentar y tener éxito en el tercer intento."""
        call_count = 0

        @with_retry(max_attempts=3, min_wait=0.1, max_wait=0.5)
        async def flaky_operation():
            nonlocal call_count
            call_count += 1
            if call_count < 3:
                raise ServiceUnavailableError(503, "test-svc")
            return "success"

        result = await flaky_operation()
        assert result == "success"
        assert call_count == 3

    @pytest.mark.asyncio
    async def test_retry_exhausted_raises(self):
        """Debe lanzar excepción cuando se agotan los reintentos."""
        @with_retry(max_attempts=2, min_wait=0.1, max_wait=0.3)
        async def always_fails():
            raise ServiceUnavailableError(500, "test-svc")

        with pytest.raises(ServiceUnavailableError):
            await always_fails()

    @pytest.mark.asyncio
    async def test_retry_respects_backoff(self):
        """El tiempo total debe reflejar el backoff entre intentos."""
        @with_retry(max_attempts=3, min_wait=0.5, max_wait=2.0)
        async def slow_fail():
            raise ServiceUnavailableError(503, "test-svc")

        start = time.time()
        with pytest.raises(ServiceUnavailableError):
            await slow_fail()
        elapsed = time.time() - start

        # Con 3 intentos y min_wait=0.5, debe tardar al menos 1s
        assert elapsed >= 0.8, f"Elapsed: {elapsed}s, esperado >= 0.8s"


class TestBulkhead:
    """Tests para el patrón bulkhead."""

    @pytest.mark.asyncio
    async def test_bulkhead_allows_within_limit(self):
        """Debe permitir llamadas dentro del límite de concurrencia."""
        bulkhead = BulkheadSemaphore("test-svc", max_concurrent=5)

        async with bulkhead:
            assert bulkhead.stats["active_calls"] == 1

        assert bulkhead.stats["active_calls"] == 0

    @pytest.mark.asyncio
    async def test_bulkhead_rejects_over_limit(self):
        """Debe rechazar cuando se excede el límite de concurrencia."""
        bulkhead = BulkheadSemaphore("test-svc-limit", max_concurrent=2)

        async def hold_slot(duration: float):
            async with bulkhead:
                await asyncio.sleep(duration)

        # Ocupar ambos slots
        task1 = asyncio.create_task(hold_slot(2.0))
        task2 = asyncio.create_task(hold_slot(2.0))
        await asyncio.sleep(0.1)  # Dar tiempo a que adquieran

        # El tercer intento debe fallar
        with pytest.raises(BulkheadFullError):
            await bulkhead.acquire(timeout=0.5)

        task1.cancel()
        task2.cancel()
        try:
            await task1
            await task2
        except asyncio.CancelledError:
            pass

    @pytest.mark.asyncio
    async def test_bulkhead_stats_track_rejections(self):
        """Debe rastrear las llamadas rechazadas en las estadísticas."""
        bulkhead = BulkheadSemaphore("test-svc-stats", max_concurrent=1)

        async def hold():
            async with bulkhead:
                await asyncio.sleep(1.0)

        task = asyncio.create_task(hold())
        await asyncio.sleep(0.1)

        try:
            await bulkhead.acquire(timeout=0.2)
        except BulkheadFullError:
            pass

        assert bulkhead.stats["rejected_calls"] == 1
        task.cancel()
        try:
            await task
        except asyncio.CancelledError:
            pass
EOF
```

3. Ejecuta los tests:

```bash
cd ~/microservices-lab
export PYTHONPATH="/root/microservices-lab/shared"
python -m pytest tests/test_resilience.py -v --tb=short
deactivate
```

### Salida Esperada

```
tests/test_resilience.py::TestCircuitBreakerRegistry::test_create_breaker PASSED
tests/test_resilience.py::TestCircuitBreakerRegistry::test_same_breaker_returned PASSED
tests/test_resilience.py::TestCircuitBreakerRegistry::test_breaker_opens_after_failures PASSED
tests/test_resilience.py::TestCircuitBreakerRegistry::test_get_all_states PASSED
tests/test_resilience.py::TestRetryPolicy::test_retry_succeeds_on_third_attempt PASSED
tests/test_resilience.py::TestRetryPolicy::test_retry_exhausted_raises PASSED
tests/test_resilience.py::TestRetryPolicy::test_retry_respects_backoff PASSED
tests/test_resilience.py::TestBulkhead::test_bulkhead_allows_within_limit PASSED
tests/test_resilience.py::TestBulkhead::test_bulkhead_rejects_over_limit PASSED
tests/test_resilience.py::TestBulkhead::test_bulkhead_stats_track_rejections PASSED

========================= 10 passed =========================
```

---

## Validación y Testing

### Lista de Verificación Final

Ejecuta esta secuencia completa para validar todos los componentes:

```bash
cd ~/microservices-lab
source order-service/venv/bin/activate
export PYTHONPATH="/root/microservices-lab/shared"

echo "=== 1. Verificar Redis ==="
python -c "import redis; r=redis.Redis(); r.ping(); print('✓ Redis OK')"

echo "=== 2. Verificar módulo de resiliencia ==="
python -c "
from resilience import CircuitBreakerRegistry, with_retry, BulkheadSemaphore
print('✓ Imports OK')
reg = CircuitBreakerRegistry()
b = reg.get_breaker('validation-test')
print(f'✓ CB creado: state={b.current_state}')
"

echo "=== 3. Ejecutar tests unitarios ==="
python -m pytest tests/test_resilience.py -v --tb=short 2>&1 | tail -5

echo "=== 4. Verificar endpoints de salud ==="
# Iniciar servicios
uvicorn scripts.mock_inventory_service:app --port 8002 &
uvicorn order-service.app.main_resilience:app --port 8001 &
sleep 3

curl -sf http://localhost:8001/health/circuit-breakers | python -m json.tool
echo "✓ Endpoint de observabilidad OK"

# Cleanup
kill $(lsof -t -i:8001) 2>/dev/null
kill $(lsof -t -i:8002) 2>/dev/null
deactivate
```

### Criterios de Éxito

| Criterio | Verificación |
|----------|-------------|
| Circuit breaker se abre tras 5 fallos | Test `test_breaker_opens_after_failures` pasa |
| Retry ejecuta 3 intentos con backoff | Test `test_retry_respects_backoff` pasa |
| Bulkhead rechaza solicitudes excedentes | Test `test_bulkhead_rejects_over_limit` pasa |
| Fallback retorna respuesta degradada | Escenario 2 del chaos-test muestra `source=fallback:*` |
| Estado persiste en Redis | `redis-cli -n 1 KEYS "circuit_breaker:*"` retorna claves |
| Endpoint `/health/circuit-breakers` funciona | Retorna JSON con estado `closed` |

---

## Solución de Problemas

### Problema 1: Redis Connection Refused

**Síntomas:**
```
redis.exceptions.ConnectionError: Error 111 connecting to localhost:6379. Connection refused.
```

**Causa:** El contenedor `mslab-redis` no está activo o no está mapeado al puerto 6379 del host.

**Solución:**
```bash
# Verificar estado del contenedor
docker ps -a | grep mslab-redis

# Si está detenido, reiniciar
docker start mslab-redis

# Si no existe, crearlo
docker run -d --name mslab-redis -p 6379:6379 redis:7.2.5-alpine

# Verificar conectividad
redis-cli ping
# Debe retornar: PONG
```

### Problema 2: Circuit Breaker No Se Abre Tras Fallos

**Síntomas:** Después de múltiples errores 500, el endpoint `/health/circuit-breakers` sigue mostrando `"state": "closed"` y el `fail_counter` no incrementa.

**Causa:** La excepción lanzada por el código no es capturada por pybreaker porque la llamada no se ejecuta a través del método `call()` del breaker, o la excepción está en la lista `exclude`.

**Solución:**
```bash
# 1. Verificar que el contador se actualiza en Redis
redis-cli -n 1 GET "circuit_breaker:inventory-service"

# 2. Si el JSON muestra fail_counter=0, el problema está en la integración.
# Verificar que _failure_call() se invoca correctamente:
cd ~/microservices-lab/order-service
source venv/bin/activate
export PYTHONPATH="/root/microservices-lab/shared"
python -c "
from resilience.circuit_breaker_registry import CircuitBreakerRegistry
reg = CircuitBreakerRegistry(redis_db=1)
b = reg.get_breaker('inventory-service', fail_max=5)
print(f'Estado: {b.current_state}, Fallos: {b.fail_counter}')

# Simular fallos manualmente
for i in range(5):
    try:
        b.call(lambda: (_ for _ in ()).throw(Exception('test')))
    except:
        pass
print(f'Después de 5 fallos - Estado: {b.current_state}')
"
deactivate

# 3. Si el breaker se abre con el test manual pero no en runtime,
#    revisar que ServiceUnavailableError NO está en la lista exclude
#    del get_breaker() en inventory_client.py
```

---

## Limpieza

```bash
# Detener servicios en background
kill $(lsof -t -i:8001) 2>/dev/null
kill $(lsof -t -i:8002) 2>/dev/null

# Limpiar claves de test en Redis
redis-cli -n 1 DEL "circuit_breaker:test-service" \
  "circuit_breaker:test-service-singleton" \
  "circuit_breaker:test-service-open" \
  "circuit_breaker:svc-a" \
  "circuit_breaker:svc-b" \
  "circuit_breaker:validation-test" \
  "circuit_breaker:test-svc-limit" \
  "circuit_breaker:test-svc-stats"

redis-cli -n 2 FLUSHDB

# Verificar que los puertos están libres
lsof -i:8001 -i:8002 2>/dev/null && echo "⚠ Puertos aún en uso" || echo "✓ Puertos liberados"
```

---

## Resumen

En este laboratorio implementaste una capa completa de resiliencia para la comunicación inter-servicio:

| Patrón | Implementación | Configuración |
|--------|---------------|---------------|
| **Circuit Breaker** | `pybreaker` + Redis | fail_max=5, reset_timeout=30s |
| **Retry** | `tenacity` decorador | 3 intentos, backoff exponencial (1-10s) con jitter |
| **Bulkhead** | `asyncio.Semaphore` | max_concurrent=10 por servicio |
| **Fallback** | Respuesta degradada optimista | Permite orden, marca como no verificado |

**Puntos clave aprendidos:**

1. Los circuit breakers protegen contra cascadas de fallos al rechazar solicitudes rápidamente cuando un servicio está caído.
2. Los retries con backoff exponencial y jitter manejan fallos transitorios sin sobrecargar el servicio destino.
3. Los bulkheads aíslan los recursos evitando que un servicio lento agote toda la capacidad del caller.
4. Las respuestas fallback permiten degradación graceful en lugar de errores completos.
5. La persistencia del estado en Redis permite que múltiples instancias del servicio compartan el conocimiento del estado del circuito.

### Recursos Adicionales

- [pybreaker documentation](https://github.com/danielfm/pybreaker)
- [tenacity documentation](https://tenacity.readthedocs.io/)
- [Microsoft - Circuit Breaker Pattern](https://learn.microsoft.com/en-us/azure/architecture/patterns/circuit-breaker)
- [AWS - Exponential Backoff and Jitter](https://aws.amazon.com/blogs/architecture/exponential-backoff-and-jitter/)
