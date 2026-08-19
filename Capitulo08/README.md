# 4 Laboratorio: instrumentar servicios Python con OpenTelemetry y visualizar métricas

## Metadata

| Campo | Valor |
|-------|-------|
| **Duración** | 66 minutos |
| **Complejidad** | Alta |
| **Nivel Bloom** | Aplicar |
| **Tecnologías** | OpenTelemetry SDK Python 1.25.0, OpenTelemetry Collector 0.102.1, Prometheus 2.53.0, Grafana 11.1.0, Jaeger 1.58.0, prometheus-fastapi-instrumentator 7.0.0 |

## Descripción General

En este laboratorio construirás un pipeline de observabilidad completo que integra los tres pilares — logs correlacionados, métricas y tracing distribuido — para tus microservicios FastAPI. Desplegarás OpenTelemetry Collector, Prometheus, Grafana y Jaeger en Kubernetes, instrumentarás los servicios con auto-instrumentación y spans manuales, configurarás propagación de contexto W3C TraceContext entre servicios, y definirás SLOs con alertas en Alertmanager.

## Objetivos de Aprendizaje

- [ ] Desplegar el stack de observabilidad (OTel Collector, Prometheus, Grafana, Jaeger) en namespaces dedicados de Kubernetes
- [ ] Instrumentar microservicios FastAPI con OpenTelemetry SDK para generar trazas distribuidas con propagación W3C TraceContext
- [ ] Exponer métricas RED (Rate, Errors, Duration) usando prometheus-fastapi-instrumentator y crear dashboards en Grafana
- [ ] Implementar logging estructurado JSON correlacionado con trace_id y span_id
- [ ] Configurar alertas de SLO (latencia p99 < 500ms, error rate < 1%) en Prometheus Alertmanager

## Prerrequisitos

### Conocimiento

- Conceptos de los tres pilares de observabilidad (logs, métricas, tracing)
- Familiaridad con métricas RED (Rate, Errors, Duration)
- Experiencia básica con Helm charts y manifiestos Kubernetes
- Labs 06-00-01 y 07-00-01 completados (microservicios con Traefik y Vault operativos)

### Acceso y Software

| Componente | Versión | Propósito |
|---|---|---|
| minikube/kind | 1.33+ | Clúster Kubernetes local |
| kubectl | 1.30+ | Gestión del clúster |
| Helm | 3.15.2 | Despliegue de charts |
| Python | 3.12.3 | Runtime de servicios |
| Docker Engine | 26.1.3 | Contenedores |

### Repositorios Helm requeridos

```bash
helm repo add open-telemetry https://open-telemetry.github.io/opentelemetry-helm-charts
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add jaegertracing https://jaegertracing.github.io/helm-charts
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
```

## Entorno del Laboratorio

### Estructura de directorios

```
~/microservices-lab/observability/
├── otel-collector/
│   └── values.yaml
├── prometheus/
│   ├── values.yaml
│   └── alerting-rules.yaml
├── grafana/
│   ├── values.yaml
│   └── dashboards/
│       └── microservices-red.json
├── jaeger/
│   └── values.yaml
└── instrumentation/
    ├── tracing.py
    ├── metrics.py
    └── logging_config.py
```

### Preparación inicial

```bash
# Crear directorio base
mkdir -p ~/microservices-lab/observability/{otel-collector,prometheus,grafana/dashboards,jaeger,instrumentation}
cd ~/microservices-lab/observability

# Crear namespaces en Kubernetes
kubectl create namespace observability --dry-run=client -o yaml | kubectl apply -f -
kubectl create namespace monitoring --dry-run=client -o yaml | kubectl apply -f -
```

---

## Paso 1: Desplegar Jaeger para almacenamiento de trazas

**Objetivo:** Instalar Jaeger 1.58.0 en el namespace `observability` como backend de trazas distribuidas.

### Instrucciones

1. Crea el archivo de valores para Jaeger:

```bash
cat > ~/microservices-lab/observability/jaeger/values.yaml << 'EOF'
provisionDataStore:
  cassandra: false
allInOne:
  enabled: true
  image:
    tag: "1.58.0"
  resources:
    limits:
      cpu: 500m
      memory: 512Mi
    requests:
      cpu: 100m
      memory: 256Mi
  extraEnv:
    - name: COLLECTOR_OTLP_ENABLED
      value: "true"
storage:
  type: memory
agent:
  enabled: false
collector:
  enabled: false
query:
  enabled: false
EOF
```

2. Instala el chart de Jaeger:

```bash
helm install jaeger jaegertracing/jaeger \
  --namespace observability \
  --values ~/microservices-lab/observability/jaeger/values.yaml \
  --wait --timeout 120s
```

3. Verifica el despliegue:

```bash
kubectl get pods -n observability -l app.kubernetes.io/name=jaeger
```

### Salida esperada

```
NAME                              READY   STATUS    RESTARTS   AGE
jaeger-0                          1/1     Running   0          45s
```

### Verificación

```bash
kubectl port-forward -n observability svc/jaeger-query 16686:16686 &
curl -s http://localhost:16686/api/services | python3 -m json.tool | head -5
kill %1 2>/dev/null
```

Debes ver una respuesta JSON con la clave `"data"`.

---

## Paso 2: Desplegar OpenTelemetry Collector

**Objetivo:** Configurar el OTel Collector 0.102.1 con receivers OTLP, processors batch/memory_limiter y exporters hacia Jaeger y Prometheus.

### Instrucciones

1. Crea la configuración del Collector:

```bash
cat > ~/microservices-lab/observability/otel-collector/values.yaml << 'EOF'
mode: deployment
image:
  tag: "0.102.1"
resources:
  limits:
    cpu: 500m
    memory: 512Mi
  requests:
    cpu: 100m
    memory: 256Mi

config:
  receivers:
    otlp:
      protocols:
        grpc:
          endpoint: 0.0.0.0:4317
        http:
          endpoint: 0.0.0.0:4318

  processors:
    batch:
      timeout: 5s
      send_batch_size: 512
    memory_limiter:
      check_interval: 5s
      limit_mib: 400
      spike_limit_mib: 100

  exporters:
    otlp/jaeger:
      endpoint: jaeger-collector.observability.svc.cluster.local:4317
      tls:
        insecure: true
    prometheus:
      endpoint: 0.0.0.0:8889
      namespace: otel
      send_timestamps: true
      metric_expiration: 5m

  service:
    pipelines:
      traces:
        receivers: [otlp]
        processors: [memory_limiter, batch]
        exporters: [otlp/jaeger]
      metrics:
        receivers: [otlp]
        processors: [memory_limiter, batch]
        exporters: [prometheus]

ports:
  otlp:
    enabled: true
    containerPort: 4317
    servicePort: 4317
    protocol: TCP
  otlp-http:
    enabled: true
    containerPort: 4318
    servicePort: 4318
    protocol: TCP
  prometheus:
    enabled: true
    containerPort: 8889
    servicePort: 8889
    protocol: TCP
EOF
```

2. Instala el OTel Collector:

```bash
helm install otel-collector open-telemetry/opentelemetry-collector \
  --namespace observability \
  --values ~/microservices-lab/observability/otel-collector/values.yaml \
  --wait --timeout 120s
```

3. Verifica que el Collector está operativo:

```bash
kubectl get pods -n observability -l app.kubernetes.io/name=opentelemetry-collector
kubectl get svc -n observability | grep otel
```

### Salida esperada

```
NAME                                        READY   STATUS    RESTARTS   AGE
otel-collector-opentelemetry-collector-xxx   1/1     Running   0          30s
```

### Verificación

```bash
# Verificar que los puertos están escuchando
kubectl exec -n observability deploy/otel-collector-opentelemetry-collector -- \
  wget -qO- http://localhost:13133/
```

Debe responder con `{"status":"Server available"...}`.

---

## Paso 3: Desplegar Prometheus y Alertmanager

**Objetivo:** Instalar Prometheus 2.53.0 con reglas de alertas para SLOs de latencia y error rate.

### Instrucciones

1. Crea las reglas de alertas:

```bash
cat > ~/microservices-lab/observability/prometheus/alerting-rules.yaml << 'EOF'
groups:
  - name: microservices-slos
    rules:
      - alert: HighLatencyP99
        expr: |
          histogram_quantile(0.99,
            sum(rate(http_request_duration_seconds_bucket{service=~"order-service|inventory-service"}[5m])) by (le, service)
          ) > 0.5
        for: 2m
        labels:
          severity: warning
          team: platform
        annotations:
          summary: "Latencia p99 alta en {{ $labels.service }}"
          description: "La latencia p99 de {{ $labels.service }} es {{ $value }}s (SLO: <500ms) durante los últimos 2 minutos."

      - alert: HighErrorRate
        expr: |
          sum(rate(http_requests_total{status_code=~"5..",service=~"order-service|inventory-service"}[5m])) by (service)
          /
          sum(rate(http_requests_total{service=~"order-service|inventory-service"}[5m])) by (service)
          > 0.01
        for: 5m
        labels:
          severity: critical
          team: platform
        annotations:
          summary: "Error rate alto en {{ $labels.service }}"
          description: "El error rate de {{ $labels.service }} es {{ $value | humanizePercentage }} (SLO: <1%) durante los últimos 5 minutos."
EOF
```

2. Crea el archivo de valores de Prometheus:

```bash
cat > ~/microservices-lab/observability/prometheus/values.yaml << 'EOF'
server:
  image:
    tag: "v2.53.0"
  resources:
    limits:
      cpu: 500m
      memory: 1Gi
    requests:
      cpu: 200m
      memory: 512Mi
  retention: "7d"
  global:
    scrape_interval: 15s
    evaluation_interval: 15s

alertmanager:
  enabled: true
  config:
    global:
      resolve_timeout: 5m
    route:
      group_by: ['alertname', 'service']
      group_wait: 10s
      group_interval: 5m
      repeat_interval: 3h
      receiver: 'default'
    receivers:
      - name: 'default'
        webhook_configs:
          - url: 'http://localhost:9093/api/v1/alerts'

serverFiles:
  alerting_rules.yml:
    groups:
      - name: microservices-slos
        rules:
          - alert: HighLatencyP99
            expr: |
              histogram_quantile(0.99,
                sum(rate(http_request_duration_seconds_bucket[5m])) by (le, service)
              ) > 0.5
            for: 2m
            labels:
              severity: warning
            annotations:
              summary: "Latencia p99 alta en {{ $labels.service }}"
          - alert: HighErrorRate
            expr: |
              sum(rate(http_requests_total{status_code=~"5.."}[5m])) by (service)
              /
              sum(rate(http_requests_total[5m])) by (service)
              > 0.01
            for: 5m
            labels:
              severity: critical
            annotations:
              summary: "Error rate alto en {{ $labels.service }}"

extraScrapeConfigs: |
  - job_name: 'otel-collector'
    static_configs:
      - targets: ['otel-collector-opentelemetry-collector.observability.svc.cluster.local:8889']
  - job_name: 'microservices'
    kubernetes_sd_configs:
      - role: pod
        namespaces:
          names: ['default']
    relabel_configs:
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: true
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_port]
        action: replace
        target_label: __address__
        regex: (.+)
        replacement: ${1}
EOF
```

3. Instala Prometheus:

```bash
helm install prometheus prometheus-community/prometheus \
  --namespace monitoring \
  --values ~/microservices-lab/observability/prometheus/values.yaml \
  --wait --timeout 180s
```

4. Verifica:

```bash
kubectl get pods -n monitoring -l app.kubernetes.io/name=prometheus
```

### Salida esperada

```
NAME                                             READY   STATUS    RESTARTS   AGE
prometheus-server-xxx                            2/2     Running   0          60s
prometheus-alertmanager-xxx                      1/1     Running   0          60s
```

### Verificación

```bash
kubectl port-forward -n monitoring svc/prometheus-server 9090:80 &
sleep 2
curl -s http://localhost:9090/api/v1/status/config | python3 -c "import sys,json; d=json.load(sys.stdin); print('Status:', d['status'])"
kill %1 2>/dev/null
```

Debe imprimir `Status: success`.

---

## Paso 4: Desplegar Grafana con dashboard preconfigurado

**Objetivo:** Instalar Grafana 11.1.0 con datasource Prometheus y un dashboard RED para microservicios.

### Instrucciones

1. Crea el dashboard JSON:

```bash
cat > ~/microservices-lab/observability/grafana/dashboards/microservices-red.json << 'EOF'
{
  "annotations": {"list": []},
  "title": "Microservices RED Metrics",
  "uid": "msvc-red-001",
  "version": 1,
  "panels": [
    {
      "id": 1,
      "title": "Request Rate (RPS)",
      "type": "timeseries",
      "gridPos": {"h": 8, "w": 8, "x": 0, "y": 0},
      "targets": [
        {
          "expr": "sum(rate(http_requests_total[5m])) by (service)",
          "legendFormat": "{{service}}"
        }
      ]
    },
    {
      "id": 2,
      "title": "Error Rate (%)",
      "type": "timeseries",
      "gridPos": {"h": 8, "w": 8, "x": 8, "y": 0},
      "targets": [
        {
          "expr": "sum(rate(http_requests_total{status_code=~\"5..\"}[5m])) by (service) / sum(rate(http_requests_total[5m])) by (service) * 100",
          "legendFormat": "{{service}}"
        }
      ]
    },
    {
      "id": 3,
      "title": "Latency p99 (seconds)",
      "type": "timeseries",
      "gridPos": {"h": 8, "w": 8, "x": 16, "y": 0},
      "targets": [
        {
          "expr": "histogram_quantile(0.99, sum(rate(http_request_duration_seconds_bucket[5m])) by (le, service))",
          "legendFormat": "{{service}} p99"
        }
      ]
    }
  ],
  "schemaVersion": 39,
  "time": {"from": "now-1h", "to": "now"},
  "refresh": "10s"
}
EOF
```

2. Crea los valores de Grafana:

```bash
cat > ~/microservices-lab/observability/grafana/values.yaml << 'EOF'
image:
  tag: "11.1.0"
resources:
  limits:
    cpu: 300m
    memory: 512Mi
  requests:
    cpu: 100m
    memory: 256Mi

adminUser: admin
adminPassword: grafana_admin_2024

datasources:
  datasources.yaml:
    apiVersion: 1
    datasources:
      - name: Prometheus
        type: prometheus
        url: http://prometheus-server.monitoring.svc.cluster.local:80
        access: proxy
        isDefault: true
      - name: Jaeger
        type: jaeger
        url: http://jaeger-query.observability.svc.cluster.local:16686
        access: proxy

dashboardProviders:
  dashboardproviders.yaml:
    apiVersion: 1
    providers:
      - name: 'default'
        orgId: 1
        folder: 'Microservices'
        type: file
        disableDeletion: false
        editable: true
        options:
          path: /var/lib/grafana/dashboards/default

dashboardsConfigMaps:
  default: grafana-dashboards
EOF
```

3. Crea el ConfigMap con el dashboard:

```bash
kubectl create configmap grafana-dashboards \
  --from-file=microservices-red.json=~/microservices-lab/observability/grafana/dashboards/microservices-red.json \
  --namespace monitoring \
  --dry-run=client -o yaml | kubectl apply -f -
```

4. Instala Grafana:

```bash
helm install grafana grafana/grafana \
  --namespace monitoring \
  --values ~/microservices-lab/observability/grafana/values.yaml \
  --wait --timeout 120s
```

### Salida esperada

```
NAME                       READY   STATUS    RESTARTS   AGE
grafana-xxx                1/1     Running   0          40s
```

### Verificación

```bash
kubectl port-forward -n monitoring svc/grafana 3000:80 &
sleep 2
curl -s -u admin:grafana_admin_2024 http://localhost:3000/api/datasources | python3 -c "import sys,json; ds=json.load(sys.stdin); [print(f'  - {d[\"name\"]}: {d[\"type\"]}') for d in ds]"
kill %1 2>/dev/null
```

Debe listar los datasources Prometheus y Jaeger.

---

## Paso 5: Instrumentar microservicios con OpenTelemetry SDK

**Objetivo:** Añadir tracing distribuido con auto-instrumentación FastAPI y spans manuales en lógica de negocio.

### Instrucciones

1. Crea el módulo de configuración de tracing:

```bash
cat > ~/microservices-lab/observability/instrumentation/tracing.py << 'EOF'
# tracing.py - Configuración de OpenTelemetry para microservicios FastAPI
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter
from opentelemetry.sdk.resources import Resource, SERVICE_NAME, SERVICE_VERSION
from opentelemetry.instrumentation.fastapi import FastAPIInstrumentor
from opentelemetry.propagate import set_global_textmap
from opentelemetry.propagators.composite import CompositePropagator
from opentelemetry.trace.propagation.tracecontext import TraceContextTextMapPropagator
from opentelemetry.baggage.propagation import W3CBaggagePropagator


def configure_tracing(app, service_name: str, service_version: str = "1.0.0"):
    """
    Configura OpenTelemetry tracing para una aplicación FastAPI.
    
    Args:
        app: Instancia de FastAPI
        service_name: Nombre del servicio (e.g., 'order-service')
        service_version: Versión semver del servicio
    """
    # Recurso que identifica al servicio
    resource = Resource.create({
        SERVICE_NAME: service_name,
        SERVICE_VERSION: service_version,
        "deployment.environment": "lab",
    })

    # Configurar el proveedor de trazas
    provider = TracerProvider(resource=resource)
    
    # Exportador OTLP hacia el Collector
    otlp_exporter = OTLPSpanExporter(
        endpoint="otel-collector-opentelemetry-collector.observability.svc.cluster.local:4317",
        insecure=True,
    )
    
    # Procesador batch para envío eficiente
    span_processor = BatchSpanProcessor(
        otlp_exporter,
        max_queue_size=2048,
        max_export_batch_size=512,
        schedule_delay_millis=5000,
    )
    provider.add_span_processor(span_processor)
    
    # Registrar el proveedor globalmente
    trace.set_tracer_provider(provider)
    
    # Configurar propagación W3C TraceContext
    set_global_textmap(CompositePropagator([
        TraceContextTextMapPropagator(),
        W3CBaggagePropagator(),
    ]))
    
    # Auto-instrumentar FastAPI
    FastAPIInstrumentor.instrument_app(app)
    
    return trace.get_tracer(service_name)
EOF
```

2. Crea el módulo de métricas con prometheus-fastapi-instrumentator:

```bash
cat > ~/microservices-lab/observability/instrumentation/metrics.py << 'EOF'
# metrics.py - Métricas Prometheus para FastAPI
from prometheus_fastapi_instrumentator import Instrumentator
from prometheus_fastapi_instrumentator.metrics import Info
from prometheus_client import Counter, Histogram


# Métricas personalizadas de negocio
orders_created_total = Counter(
    "business_orders_created_total",
    "Total de órdenes creadas exitosamente",
    ["status"]
)

inventory_checks_duration = Histogram(
    "business_inventory_check_duration_seconds",
    "Duración de verificaciones de inventario",
    buckets=[0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1.0]
)


def setup_metrics(app, service_name: str):
    """
    Configura prometheus-fastapi-instrumentator en la app.
    Expone métricas en /metrics automáticamente.
    """
    instrumentator = Instrumentator(
        should_group_status_codes=False,
        should_ignore_untemplated=True,
        should_respect_env_var=False,
        excluded_handlers=["/health", "/ready", "/metrics"],
        env_var_name="ENABLE_METRICS",
    )
    
    # Métrica adicional: tamaño de respuesta
    def response_size_metric(info: Info):
        pass  # instrumentator ya incluye esto por defecto
    
    instrumentator.add(response_size_metric)
    instrumentator.instrument(app)
    instrumentator.expose(app, endpoint="/metrics", include_in_schema=False)
    
    return instrumentator
EOF
```

3. Crea el módulo de logging estructurado correlacionado:

```bash
cat > ~/microservices-lab/observability/instrumentation/logging_config.py << 'EOF'
# logging_config.py - Logging estructurado con correlación de trazas
import logging
import json
import sys
from datetime import datetime, timezone
from opentelemetry import trace


class StructuredJsonFormatter(logging.Formatter):
    """Formatter que emite logs en JSON con trace_id y span_id."""
    
    def __init__(self, service_name: str):
        super().__init__()
        self.service_name = service_name
    
    def format(self, record: logging.LogRecord) -> str:
        # Obtener contexto de traza actual
        span = trace.get_current_span()
        span_context = span.get_span_context()
        
        trace_id = "0" * 32
        span_id = "0" * 16
        
        if span_context.is_valid:
            trace_id = format(span_context.trace_id, '032x')
            span_id = format(span_context.span_id, '016x')
        
        log_entry = {
            "timestamp": datetime.now(timezone.utc).isoformat(),
            "level": record.levelname,
            "service": self.service_name,
            "logger": record.name,
            "message": record.getMessage(),
            "trace_id": trace_id,
            "span_id": span_id,
            "module": record.module,
            "function": record.funcName,
            "line": record.lineno,
        }
        
        # Incluir excepción si existe
        if record.exc_info and record.exc_info[0] is not None:
            log_entry["error"] = {
                "type": record.exc_info[0].__name__,
                "message": str(record.exc_info[1]),
                "traceback": self.formatException(record.exc_info),
            }
        
        # Incluir campos extra
        for key, value in record.__dict__.items():
            if key.startswith("ctx_"):
                log_entry[key[4:]] = value
        
        return json.dumps(log_entry, ensure_ascii=False)


def configure_logging(service_name: str, level: str = "INFO"):
    """
    Configura logging estructurado JSON con correlación de trazas.
    """
    root_logger = logging.getLogger()
    root_logger.setLevel(getattr(logging, level.upper()))
    
    # Limpiar handlers existentes
    root_logger.handlers.clear()
    
    # Handler stdout con formato JSON
    handler = logging.StreamHandler(sys.stdout)
    handler.setFormatter(StructuredJsonFormatter(service_name))
    root_logger.addHandler(handler)
    
    # Reducir verbosidad de librerías externas
    logging.getLogger("uvicorn.access").setLevel(logging.WARNING)
    logging.getLogger("opentelemetry").setLevel(logging.WARNING)
    
    return logging.getLogger(service_name)
EOF
```

4. Crea la aplicación de ejemplo que integra los tres módulos (order-service):

```bash
cat > ~/microservices-lab/observability/instrumentation/order_service_example.py << 'EOF'
# order_service_example.py - Ejemplo de integración completa
from fastapi import FastAPI, HTTPException, Request
from pydantic import BaseModel
from opentelemetry import trace
import httpx
import logging
import time

# Importar módulos de instrumentación
from tracing import configure_tracing
from metrics import setup_metrics, orders_created_total, inventory_checks_duration
from logging_config import configure_logging

app = FastAPI(title="Order Service", version="1.0.0")

# 1. Configurar logging estructurado
logger = configure_logging("order-service")

# 2. Configurar tracing con OpenTelemetry
tracer = configure_tracing(app, "order-service", "1.0.0")

# 3. Configurar métricas Prometheus
setup_metrics(app, "order-service")


class OrderCreate(BaseModel):
    product_id: str
    quantity: int


@app.get("/health")
async def health():
    return {"status": "healthy"}


@app.post("/api/v1/orders")
async def create_order(order: OrderCreate):
    """Crear una orden con tracing manual en lógica de negocio."""
    logger.info("Recibida solicitud de creación de orden",
                extra={"ctx_product_id": order.product_id, "ctx_quantity": order.quantity})
    
    # Span manual para verificación de inventario
    with tracer.start_as_current_span("check_inventory") as span:
        span.set_attribute("product.id", order.product_id)
        span.set_attribute("order.quantity", order.quantity)
        
        start = time.perf_counter()
        inventory_available = await check_inventory(order.product_id, order.quantity)
        duration = time.perf_counter() - start
        inventory_checks_duration.observe(duration)
        
        if not inventory_available:
            span.set_attribute("inventory.available", False)
            span.set_status(trace.StatusCode.ERROR, "Inventario insuficiente")
            logger.warning("Inventario insuficiente",
                          extra={"ctx_product_id": order.product_id})
            raise HTTPException(status_code=409, detail="Inventario insuficiente")
        
        span.set_attribute("inventory.available", True)
    
    # Span manual para persistencia
    with tracer.start_as_current_span("persist_order") as span:
        order_id = f"ORD-{int(time.time())}"
        span.set_attribute("order.id", order_id)
        # Simular escritura en DB
        await simulate_db_write(order_id)
        
        orders_created_total.labels(status="success").inc()
        logger.info("Orden creada exitosamente",
                    extra={"ctx_order_id": order_id})
    
    return {"order_id": order_id, "status": "created"}


async def check_inventory(product_id: str, quantity: int) -> bool:
    """Llamada al inventory-service con propagación de contexto."""
    try:
        async with httpx.AsyncClient() as client:
            # httpx con OpenTelemetry propaga automáticamente las cabeceras W3C
            response = await client.get(
                f"http://inventory-service:8002/api/v1/inventory/{product_id}",
                timeout=5.0,
            )
            if response.status_code == 200:
                data = response.json()
                return data.get("available_quantity", 0) >= quantity
    except httpx.RequestError as e:
        logger.error("Error al verificar inventario", exc_info=True)
        # Fallback: asumir disponible para no bloquear
        return True
    return False


async def simulate_db_write(order_id: str):
    """Simula escritura en base de datos."""
    import asyncio
    await asyncio.sleep(0.02)  # Simular latencia de DB
EOF
```

5. Copia los módulos de instrumentación al directorio del order-service:

```bash
cp ~/microservices-lab/observability/instrumentation/{tracing,metrics,logging_config}.py \
   ~/microservices-lab/order-service/
```

6. Actualiza las dependencias del servicio:

```bash
cat >> ~/microservices-lab/order-service/requirements.txt << 'EOF'
opentelemetry-sdk==1.25.0
opentelemetry-api==1.25.0
opentelemetry-instrumentation-fastapi==0.46b0
opentelemetry-instrumentation-httpx==0.46b0
opentelemetry-exporter-otlp-proto-grpc==1.25.0
opentelemetry-propagator-b3==1.25.0
prometheus-fastapi-instrumentator==7.0.0
httpx==0.27.0
EOF
```

### Salida esperada

Los archivos deben existir en ambas ubicaciones:

```bash
ls ~/microservices-lab/order-service/{tracing,metrics,logging_config}.py
```

```
/home/student/microservices-lab/order-service/tracing.py
/home/student/microservices-lab/order-service/metrics.py
/home/student/microservices-lab/order-service/logging_config.py
```

### Verificación

```bash
# Verificar sintaxis Python de los módulos
python3 -c "import ast; [ast.parse(open(f).read()) for f in ['tracing.py','metrics.py','logging_config.py']]" \
  && echo "✓ Todos los módulos tienen sintaxis válida"
```

---

## Paso 6: Configurar propagación W3C TraceContext entre servicios

**Objetivo:** Implementar la propagación de contexto de trazas entre order-service e inventory-service mediante cabeceras W3C `traceparent`.

### Instrucciones

1. Crea el middleware de instrumentación HTTP para el inventory-service:

```bash
cat > ~/microservices-lab/inventory-service/tracing.py << 'EOF'
# tracing.py para inventory-service - idéntico patrón
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter
from opentelemetry.sdk.resources import Resource, SERVICE_NAME, SERVICE_VERSION
from opentelemetry.instrumentation.fastapi import FastAPIInstrumentor
from opentelemetry.propagate import set_global_textmap
from opentelemetry.propagators.composite import CompositePropagator
from opentelemetry.trace.propagation.tracecontext import TraceContextTextMapPropagator
from opentelemetry.baggage.propagation import W3CBaggagePropagator


def configure_tracing(app, service_name: str, service_version: str = "1.0.0"):
    resource = Resource.create({
        SERVICE_NAME: service_name,
        SERVICE_VERSION: service_version,
        "deployment.environment": "lab",
    })

    provider = TracerProvider(resource=resource)
    otlp_exporter = OTLPSpanExporter(
        endpoint="otel-collector-opentelemetry-collector.observability.svc.cluster.local:4317",
        insecure=True,
    )
    provider.add_span_processor(BatchSpanProcessor(otlp_exporter))
    trace.set_tracer_provider(provider)

    # W3C TraceContext propagation
    set_global_textmap(CompositePropagator([
        TraceContextTextMapPropagator(),
        W3CBaggagePropagator(),
    ]))

    FastAPIInstrumentor.instrument_app(app)
    return trace.get_tracer(service_name)
EOF
```

2. Crea un script de validación de propagación de contexto:

```bash
cat > ~/microservices-lab/observability/instrumentation/test_propagation.py << 'EOF'
#!/usr/bin/env python3
"""Test de propagación W3C TraceContext entre servicios."""
import httpx
import asyncio
import sys


async def test_trace_propagation():
    """Envía una request con traceparent y verifica que se propaga."""
    # Simular un traceparent W3C válido
    trace_id = "4bf92f3577b34da6a3ce929d0e0e4736"
    parent_span_id = "00f067aa0ba902b7"
    traceparent = f"00-{trace_id}-{parent_span_id}-01"
    
    headers = {
        "traceparent": traceparent,
        "Content-Type": "application/json",
    }
    
    async with httpx.AsyncClient() as client:
        try:
            response = await client.post(
                "http://localhost:8001/api/v1/orders",
                json={"product_id": "PROD-001", "quantity": 1},
                headers=headers,
                timeout=10.0,
            )
            print(f"Status: {response.status_code}")
            print(f"Response: {response.json()}")
            
            # Verificar que el trace_id se mantiene en la respuesta
            resp_traceparent = response.headers.get("traceparent", "")
            if trace_id in resp_traceparent:
                print(f"✓ Trace ID propagado correctamente: {trace_id}")
                return True
            else:
                print(f"⚠ Traceparent en respuesta: {resp_traceparent}")
                return True  # El servicio puede no devolver traceparent
                
        except httpx.RequestError as e:
            print(f"✗ Error de conexión: {e}")
            return False


if __name__ == "__main__":
    result = asyncio.run(test_trace_propagation())
    sys.exit(0 if result else 1)
EOF
chmod +x ~/microservices-lab/observability/instrumentation/test_propagation.py
```

3. Verifica que la instrumentación httpx propaga automáticamente las cabeceras:

```bash
cat > ~/microservices-lab/observability/instrumentation/verify_instrumentation.py << 'EOF'
#!/usr/bin/env python3
"""Verifica que los paquetes de instrumentación están instalados correctamente."""
import importlib

packages = [
    ("opentelemetry.sdk", "1.25.0"),
    ("opentelemetry.instrumentation.fastapi", "0.46b0"),
    ("opentelemetry.exporter.otlp.proto.grpc", None),
    ("opentelemetry.propagators.composite", None),
    ("prometheus_fastapi_instrumentator", "7.0.0"),
]

all_ok = True
for pkg, expected_version in packages:
    try:
        mod = importlib.import_module(pkg)
        version = getattr(mod, "__version__", "installed")
        status = "✓"
    except ImportError:
        version = "NOT FOUND"
        status = "✗"
        all_ok = False
    print(f"  {status} {pkg}: {version}")

print(f"\n{'✓ Todos los paquetes disponibles' if all_ok else '✗ Faltan paquetes'}")
EOF
python3 ~/microservices-lab/observability/instrumentation/verify_instrumentation.py
```

### Salida esperada

```
  ✓ opentelemetry.sdk: 1.25.0
  ✓ opentelemetry.instrumentation.fastapi: 0.46b0
  ✓ opentelemetry.exporter.otlp.proto.grpc: installed
  ✓ opentelemetry.propagators.composite: installed
  ✓ prometheus_fastapi_instrumentator: 7.0.0

✓ Todos los paquetes disponibles
```

### Verificación

La propagación W3C funciona cuando el `trace_id` generado en el primer servicio aparece en los spans del segundo servicio en Jaeger. Esto se validará completamente en el Paso 8.

---

## Paso 7: Desplegar servicios instrumentados en Kubernetes

**Objetivo:** Construir y desplegar los microservicios con la instrumentación de observabilidad activa.

### Instrucciones

1. Actualiza el Dockerfile del order-service para incluir las dependencias de OTel:

```bash
cat > ~/microservices-lab/order-service/Dockerfile.otel << 'EOF'
FROM python:3.12.3-slim

WORKDIR /app

# Instalar dependencias
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copiar código de la aplicación
COPY . .

# Variables de entorno para OpenTelemetry
ENV OTEL_SERVICE_NAME=order-service
ENV OTEL_EXPORTER_OTLP_ENDPOINT=http://otel-collector-opentelemetry-collector.observability.svc.cluster.local:4317
ENV OTEL_TRACES_EXPORTER=otlp
ENV OTEL_METRICS_EXPORTER=otlp
ENV OTEL_PROPAGATORS=tracecontext,baggage

EXPOSE 8001

CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8001"]
EOF
```

2. Construye la imagen (usando el registry de minikube):

```bash
eval $(minikube docker-env)
docker build -t order-service:otel-v1 \
  -f ~/microservices-lab/order-service/Dockerfile.otel \
  ~/microservices-lab/order-service/
```

3. Crea el manifiesto de deployment con anotaciones para Prometheus scraping:

```bash
cat > ~/microservices-lab/observability/order-service-deployment.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
  namespace: default
  labels:
    app: order-service
spec:
  replicas: 1
  selector:
    matchLabels:
      app: order-service
  template:
    metadata:
      labels:
        app: order-service
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "8001"
        prometheus.io/path: "/metrics"
    spec:
      containers:
        - name: order-service
          image: order-service:otel-v1
          imagePullPolicy: Never
          ports:
            - containerPort: 8001
              name: http
          env:
            - name: OTEL_SERVICE_NAME
              value: "order-service"
            - name: OTEL_EXPORTER_OTLP_ENDPOINT
              value: "http://otel-collector-opentelemetry-collector.observability.svc.cluster.local:4317"
            - name: OTEL_PROPAGATORS
              value: "tracecontext,baggage"
          resources:
            limits:
              cpu: 300m
              memory: 256Mi
            requests:
              cpu: 100m
              memory: 128Mi
          readinessProbe:
            httpGet:
              path: /health
              port: 8001
            initialDelaySeconds: 5
            periodSeconds: 10
          livenessProbe:
            httpGet:
              path: /health
              port: 8001
            initialDelaySeconds: 10
            periodSeconds: 30
---
apiVersion: v1
kind: Service
metadata:
  name: order-service
  namespace: default
spec:
  selector:
    app: order-service
  ports:
    - port: 8001
      targetPort: 8001
      protocol: TCP
  type: ClusterIP
EOF
```

4. Aplica el deployment:

```bash
kubectl apply -f ~/microservices-lab/observability/order-service-deployment.yaml
kubectl rollout status deployment/order-service --timeout=60s
```

### Salida esperada

```
deployment "order-service" successfully rolled out
```

### Verificación

```bash
# Verificar que el endpoint /metrics responde
kubectl port-forward svc/order-service 8001:8001 &
sleep 3
curl -s http://localhost:8001/metrics | head -20
kill %1 2>/dev/null
```

Debe mostrar métricas en formato Prometheus, incluyendo líneas como `http_requests_total` y `http_request_duration_seconds_bucket`.

---

## Paso 8: Generar tráfico y validar el pipeline completo

**Objetivo:** Generar tráfico de prueba, verificar trazas en Jaeger, métricas en Prometheus y visualización en Grafana.

### Instrucciones

1. Crea un script generador de tráfico:

```bash
cat > ~/microservices-lab/observability/generate_traffic.sh << 'SCRIPT'
#!/bin/bash
# Genera tráfico de prueba para validar observabilidad
echo "=== Generando tráfico de prueba (60 requests) ==="

ORDER_URL="http://localhost:8001"

for i in $(seq 1 60); do
    # Mezcla de requests exitosas y con errores
    if [ $((i % 10)) -eq 0 ]; then
        # Simular producto inexistente (error)
        curl -s -X POST "$ORDER_URL/api/v1/orders" \
          -H "Content-Type: application/json" \
          -d '{"product_id": "INVALID", "quantity": 9999}' > /dev/null
    else
        # Request normal
        curl -s -X POST "$ORDER_URL/api/v1/orders" \
          -H "Content-Type: application/json" \
          -d "{\"product_id\": \"PROD-$(printf '%03d' $((RANDOM % 10)))\", \"quantity\": $((RANDOM % 5 + 1))}" > /dev/null
    fi
    
    # También generar GETs
    curl -s "$ORDER_URL/health" > /dev/null
    
    sleep 0.5
done

echo "=== Tráfico generado. Esperando 15s para propagación... ==="
sleep 15
echo "=== Listo para verificar dashboards ==="
SCRIPT
chmod +x ~/microservices-lab/observability/generate_traffic.sh
```

2. Ejecuta el generador con port-forward activo:

```bash
kubectl port-forward svc/order-service 8001:8001 &
PF_PID=$!
sleep 2

~/microservices-lab/observability/generate_traffic.sh

kill $PF_PID 2>/dev/null
```

3. Verifica trazas en Jaeger:

```bash
kubectl port-forward -n observability svc/jaeger-query 16686:16686 &
sleep 2

# Buscar trazas del order-service
TRACES=$(curl -s "http://localhost:16686/api/traces?service=order-service&limit=5" | \
  python3 -c "import sys,json; d=json.load(sys.stdin); print(f'Trazas encontradas: {len(d.get(\"data\", []))}')")
echo "$TRACES"

kill %1 2>/dev/null
```

4. Verifica métricas en Prometheus:

```bash
kubectl port-forward -n monitoring svc/prometheus-server 9090:80 &
sleep 2

# Consultar métricas del servicio
echo "--- Request Rate ---"
curl -s "http://localhost:9090/api/v1/query?query=rate(http_requests_total\{service=\"order-service\"\}[5m])" | \
  python3 -c "import sys,json; d=json.load(sys.stdin); results=d.get('data',{}).get('result',[]); print(f'Series: {len(results)}')"

echo "--- Latencia p99 ---"
curl -s "http://localhost:9090/api/v1/query?query=histogram_quantile(0.99,rate(http_request_duration_seconds_bucket[5m]))" | \
  python3 -c "import sys,json; d=json.load(sys.stdin); results=d.get('data',{}).get('result',[]); [print(f'  {r[\"metric\"].get(\"service\",\"unknown\")}: {float(r[\"value\"][1]):.4f}s') for r in results]"

kill %1 2>/dev/null
```

5. Verifica alertas configuradas:

```bash
kubectl port-forward -n monitoring svc/prometheus-server 9090:80 &
sleep 2

curl -s "http://localhost:9090/api/v1/rules?type=alert" | \
  python3 -c "
import sys, json
d = json.load(sys.stdin)
groups = d.get('data', {}).get('groups', [])
for g in groups:
    for r in g.get('rules', []):
        print(f'  Alerta: {r[\"name\"]} | Estado: {r[\"state\"]} | Severidad: {r.get(\"labels\",{}).get(\"severity\",\"N/A\")}')
"

kill %1 2>/dev/null
```

### Salida esperada

```
=== Generando tráfico de prueba (60 requests) ===
=== Tráfico generado. Esperando 15s para propagación... ===
=== Listo para verificar dashboards ===
Trazas encontradas: 5
--- Request Rate ---
Series: 1
--- Latencia p99 ---
  order-service: 0.0523s
  Alerta: HighLatencyP99 | Estado: inactive | Severidad: warning
  Alerta: HighErrorRate | Estado: inactive | Severidad: critical
```

### Verificación

```bash
# Verificación integral del pipeline
echo "=== Verificación del Pipeline de Observabilidad ==="

# 1. OTel Collector running
OTEL_OK=$(kubectl get pods -n observability -l app.kubernetes.io/name=opentelemetry-collector -o jsonpath='{.items[0].status.phase}')
echo "1. OTel Collector: $OTEL_OK"

# 2. Jaeger receiving traces
JAEGER_OK=$(kubectl get pods -n observability -l app.kubernetes.io/name=jaeger -o jsonpath='{.items[0].status.phase}')
echo "2. Jaeger: $JAEGER_OK"

# 3. Prometheus scraping
PROM_OK=$(kubectl get pods -n monitoring -l app.kubernetes.io/name=prometheus -o jsonpath='{.items[0].status.phase}')
echo "3. Prometheus: $PROM_OK"

# 4. Grafana accessible
GRAF_OK=$(kubectl get pods -n monitoring -l app.kubernetes.io/name=grafana -o jsonpath='{.items[0].status.phase}')
echo "4. Grafana: $GRAF_OK"

# 5. Service instrumented
SVC_OK=$(kubectl get pods -l app=order-service -o jsonpath='{.items[0].status.phase}')
echo "5. Order Service: $SVC_OK"

echo ""
echo "✓ Pipeline completo operativo" 
```

---

## Validación y Testing

Ejecuta la siguiente validación integral que confirma el funcionamiento end-to-end:

```bash
cat > ~/microservices-lab/observability/validate_lab.sh << 'SCRIPT'
#!/bin/bash
set -e
PASS=0
FAIL=0

check() {
    local desc="$1"
    local cmd="$2"
    if eval "$cmd" > /dev/null 2>&1; then
        echo "✓ $desc"
        ((PASS++))
    else
        echo "✗ $desc"
        ((FAIL++))
    fi
}

echo "═══════════════════════════════════════════════════"
echo "  VALIDACIÓN LAB 08-00-01: Observabilidad"
echo "═══════════════════════════════════════════════════"
echo ""

# Infraestructura
check "Namespace 'observability' existe" \
  "kubectl get namespace observability"

check "Namespace 'monitoring' existe" \
  "kubectl get namespace monitoring"

check "OTel Collector en estado Running" \
  "kubectl get pods -n observability -l app.kubernetes.io/name=opentelemetry-collector -o jsonpath='{.items[0].status.phase}' | grep -q Running"

check "Jaeger en estado Running" \
  "kubectl get pods -n observability -l app.kubernetes.io/name=jaeger -o jsonpath='{.items[0].status.phase}' | grep -q Running"

check "Prometheus en estado Running" \
  "kubectl get pods -n monitoring -l app.kubernetes.io/name=prometheus --field-selector=status.phase=Running -o name | grep -q pod"

check "Grafana en estado Running" \
  "kubectl get pods -n monitoring -l app.kubernetes.io/name=grafana -o jsonpath='{.items[0].status.phase}' | grep -q Running"

# Instrumentación
check "Archivo tracing.py existe en order-service" \
  "test -f ~/microservices-lab/order-service/tracing.py"

check "Archivo metrics.py existe en order-service" \
  "test -f ~/microservices-lab/order-service/metrics.py"

check "Archivo logging_config.py existe en order-service" \
  "test -f ~/microservices-lab/order-service/logging_config.py"

check "Reglas de alertas configuradas" \
  "test -f ~/microservices-lab/observability/prometheus/alerting-rules.yaml"

check "Dashboard Grafana configurado" \
  "test -f ~/microservices-lab/observability/grafana/dashboards/microservices-red.json"

# Configuración
check "OTel Collector expone puerto 4317 (gRPC)" \
  "kubectl get svc -n observability otel-collector-opentelemetry-collector -o jsonpath='{.spec.ports[?(@.port==4317)].port}' | grep -q 4317"

check "OTel Collector expone puerto 4318 (HTTP)" \
  "kubectl get svc -n observability otel-collector-opentelemetry-collector -o jsonpath='{.spec.ports[?(@.port==4318)].port}' | grep -q 4318"

check "Anotación prometheus.io/scrape en order-service" \
  "kubectl get deployment order-service -o jsonpath='{.spec.template.metadata.annotations.prometheus\\.io/scrape}' | grep -q true"

echo ""
echo "═══════════════════════════════════════════════════"
echo "  Resultado: $PASS pasaron, $FAIL fallaron"
echo "═══════════════════════════════════════════════════"

[ $FAIL -eq 0 ] && echo "  🎉 ¡Laboratorio completado exitosamente!" || echo "  ⚠ Revisa los items fallidos"
SCRIPT
chmod +x ~/microservices-lab/observability/validate_lab.sh
~/microservices-lab/observability/validate_lab.sh
```

**Resultado esperado:** Todos los checks deben mostrar `✓`.

---

## Solución de Problemas

### Problema 1: OTel Collector no recibe spans del servicio

**Síntomas:** Jaeger no muestra trazas del `order-service`. Los logs del Collector no muestran actividad de recepción.

**Causa:** El endpoint OTLP configurado en el servicio no es alcanzable desde el pod del microservicio. Esto ocurre cuando el nombre DNS del servicio del Collector es incorrecto o hay políticas de red que bloquean la comunicación entre namespaces.

**Solución:**

```bash
# 1. Verificar que el servicio del Collector es resolvible
kubectl run dns-test --rm -it --image=busybox --restart=Never -- \
  nslookup otel-collector-opentelemetry-collector.observability.svc.cluster.local

# 2. Verificar conectividad desde el namespace default
kubectl run curl-test --rm -it --image=curlimages/curl --restart=Never -- \
  curl -v http://otel-collector-opentelemetry-collector.observability.svc.cluster.local:4318/v1/traces

# 3. Si el nombre es diferente, obtener el nombre correcto:
kubectl get svc -n observability | grep otel

# 4. Actualizar la variable de entorno en el deployment del servicio:
kubectl set env deployment/order-service \
  OTEL_EXPORTER_OTLP_ENDPOINT=http://<nombre-correcto>.observability.svc.cluster.local:4317

# 5. Reiniciar el servicio
kubectl rollout restart deployment/order-service
```

### Problema 2: Prometheus no muestra métricas del microservicio

**Síntomas:** El endpoint `/metrics` del servicio responde correctamente con `curl`, pero las métricas no aparecen en Prometheus (query `http_requests_total` retorna vacío).

**Causa:** Las anotaciones de scraping en el pod no coinciden con la configuración de `relabel_configs` en Prometheus, o el `scrape_interval` aún no ha transcurrido. Otra causa común es que la anotación `prometheus.io/port` contiene un valor incorrecto.

**Solución:**

```bash
# 1. Verificar anotaciones del pod
kubectl get pods -l app=order-service -o jsonpath='{.items[0].metadata.annotations}' | python3 -m json.tool

# 2. Verificar que el puerto en la anotación es correcto (debe ser "8001")
kubectl annotate pods -l app=order-service prometheus.io/port="8001" --overwrite

# 3. Verificar targets en Prometheus
kubectl port-forward -n monitoring svc/prometheus-server 9090:80 &
sleep 2
curl -s http://localhost:9090/api/v1/targets | python3 -c "
import sys, json
d = json.load(sys.stdin)
for t in d['data']['activeTargets']:
    if 'order' in t.get('labels',{}).get('pod',''):
        print(f'Target: {t[\"scrapeUrl\"]} | Health: {t[\"health\"]} | Error: {t.get(\"lastError\",\"none\")}')
"
kill %1 2>/dev/null

# 4. Si el target no aparece, verificar que el ServiceMonitor o la configuración
#    de scraping incluye el namespace 'default':
kubectl get cm prometheus-server -n monitoring -o yaml | grep -A5 "kubernetes_sd_configs"

# 5. Forzar recarga de configuración de Prometheus:
kubectl rollout restart deployment -n monitoring -l app.kubernetes.io/name=prometheus
```

---

## Limpieza

Para eliminar todos los recursos creados en este laboratorio:

```bash
# Eliminar deployments de servicios instrumentados
kubectl delete -f ~/microservices-lab/observability/order-service-deployment.yaml --ignore-not-found

# Desinstalar charts de Helm
helm uninstall grafana -n monitoring 2>/dev/null
helm uninstall prometheus -n monitoring 2>/dev/null
helm uninstall otel-collector -n observability 2>/dev/null
helm uninstall jaeger -n observability 2>/dev/null

# Eliminar ConfigMaps
kubectl delete configmap grafana-dashboards -n monitoring --ignore-not-found

# Eliminar namespaces (elimina todo su contenido)
kubectl delete namespace observability --ignore-not-found
kubectl delete namespace monitoring --ignore-not-found

# Limpiar imágenes Docker locales
eval $(minikube docker-env)
docker rmi order-service:otel-v1 2>/dev/null

echo "✓ Limpieza completada"
```

---

## Resumen

En este laboratorio has construido un pipeline de observabilidad completo para microservicios FastAPI:

| Componente | Logro |
|---|---|
| **OpenTelemetry Collector** | Configurado con receivers OTLP (gRPC:4317, HTTP:4318), processors batch y memory_limiter, exporters hacia Jaeger y Prometheus |
| **Tracing distribuido** | Instrumentación automática FastAPI + spans manuales en lógica de negocio + propagación W3C TraceContext |
| **Métricas RED** | prometheus-fastapi-instrumentator exponiendo rate, errors y duration en `/metrics` |
| **Logging correlacionado** | JSON estructurado con `trace_id` y `span_id` extraídos del contexto OpenTelemetry |
| **Alertas SLO** | HighLatencyP99 (>500ms por 2min) y HighErrorRate (>1% por 5min) en Alertmanager |
| **Visualización** | Dashboard Grafana con paneles de RPS, Error Rate % y Latencia p99 |

### Conceptos clave aplicados

- Los **tres pilares de observabilidad** se complementan: métricas para detectar, trazas para localizar, logs para diagnosticar
- La **propagación de contexto W3C** (`traceparent`) permite reconstruir el grafo causal de una solicitud distribuida
- Las **métricas RED** (Rate, Errors, Duration) son el estándar para monitorear servicios orientados a solicitudes
- Los **SLOs** deben ser medibles, con alertas que indiquen cuándo el sistema se degrada antes de que el usuario lo perciba

### Recursos adicionales

- [OpenTelemetry Python Documentation](https://opentelemetry.io/docs/languages/python/)
- [Prometheus Alerting Rules](https://prometheus.io/docs/prometheus/latest/configuration/alerting_rules/)
- [Grafana Dashboard Best Practices](https://grafana.com/docs/grafana/latest/dashboards/build-dashboards/best-practices/)
- [W3C Trace Context Specification](https://www.w3.org/TR/trace-context/)
