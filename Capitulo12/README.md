# Diseñar y Ejecutar Pruebas de Carga y Ataques Controlados

## Metadatos

| Campo | Valor |
|-------|-------|
| **Duración** | 66 minutos |
| **Complejidad** | Alta |
| **Nivel Bloom** | Crear |

## Descripción General

En este laboratorio diseñarás y ejecutarás una suite completa de pruebas de carga con k6, implementarás experimentos de chaos engineering con Chaos Mesh contra los microservicios desplegados en Kubernetes, y documentarás los hallazgos en un postmortem estructurado siguiendo la plantilla Google SRE. El laboratorio valida la resiliencia del sistema midiendo el impacto real en SLOs definidos formalmente.

## Objetivos de Aprendizaje

- [ ] Diseñar y ejecutar escenarios de pruebas de carga (smoke, load, stress, spike) con k6 exportando métricas a Prometheus
- [ ] Implementar experimentos de chaos engineering controlados con Chaos Mesh (PodChaos, NetworkChaos, StressChaos)
- [ ] Definir SLOs formales con alertas en Alertmanager y medir error budgets con PromQL
- [ ] Ejecutar contract testing con schemathesis durante experimentos de caos para validar integridad de APIs
- [ ] Documentar un postmortem estructurado con timeline, root cause analysis e action items

## Prerrequisitos

### Conocimientos Requeridos

- Laboratorio 11-00-01 completado (microservicios desplegados via Argo CD)
- Familiaridad con PromQL (rate, histogram_quantile, increase)
- Conceptos SRE: SLIs, SLOs, error budgets
- Uso básico de kubectl y Helm

### Acceso Requerido

- Clúster minikube con namespaces `microservices-prod`, `monitoring` y `chaos-testing`
- Prometheus 2.53.0 y Grafana 11.1.0 operativos en namespace `monitoring`
- Chaos Mesh 2.6.3 instalado con CRDs registrados
- k6 0.51.0 instalado como binario local

## Entorno del Laboratorio

### Software Requerido

| Herramienta | Versión | Propósito |
|-------------|---------|-----------|
| k6 | 0.51.0 | Pruebas de carga con escenarios |
| Chaos Mesh | 2.6.3 | Inyección de fallos en Kubernetes |
| Prometheus | 2.53.0 | Recolección de métricas y alertas |
| Grafana | 11.1.0 | Visualización de dashboards |
| Alertmanager | 0.27.0 | Gestión de alertas |
| schemathesis | 3.27.1 | Contract testing basado en OpenAPI |
| kubectl | 1.30.2 | Gestión del clúster |

### Verificación del Entorno

```bash
# Verificar que minikube está corriendo con los namespaces necesarios
kubectl get ns microservices-prod monitoring chaos-testing

# Verificar pods de los microservicios
kubectl get pods -n microservices-prod

# Verificar Prometheus y Grafana
kubectl get pods -n monitoring -l app.kubernetes.io/name=prometheus
kubectl get pods -n monitoring -l app.kubernetes.io/name=grafana

# Verificar Chaos Mesh
kubectl get pods -n chaos-testing

# Verificar k6
k6 version

# Verificar schemathesis
pip show schemathesis
```

**Salida esperada:** Todos los pods en estado `Running`, k6 versión 0.51.0, schemathesis 3.27.1.

### Configuración de Acceso a Servicios

```bash
# Exponer servicios via port-forward (ejecutar en terminales separadas)
kubectl port-forward svc/order-service -n microservices-prod 8001:8001 &
kubectl port-forward svc/inventory-service -n microservices-prod 8002:8002 &
kubectl port-forward svc/prometheus-kube-prometheus-prometheus -n monitoring 9090:9090 &
kubectl port-forward svc/prometheus-grafana -n monitoring 3000:80 &

# Verificar conectividad
curl -s http://localhost:8001/health | jq .
curl -s http://localhost:8002/health | jq .
```

---

## Paso a Paso

### Paso 1: Crear Estructura de Directorios del Laboratorio

**Objetivo:** Establecer la estructura de archivos para scripts de carga, manifiestos de caos y documentación.

**Instrucciones:**

1. Crear la estructura de directorios:

```bash
cd ~/microservices-lab
mkdir -p tests/load
mkdir -p tests/chaos
mkdir -p tests/contracts
mkdir -p docs/runbooks
mkdir -p docs/postmortems
mkdir -p monitoring/alerts
mkdir -p monitoring/dashboards
```

2. Verificar la estructura:

```bash
find tests docs monitoring -type d | sort
```

**Salida esperada:**

```
docs/postmortems
docs/runbooks
monitoring/alerts
monitoring/dashboards
tests/chaos
tests/contracts
tests/load
```

**Verificación:** Todos los directorios existen y están vacíos, listos para recibir los artefactos del laboratorio.

---

### Paso 2: Crear Script de Smoke Test con k6

**Objetivo:** Implementar un smoke test que establezca el baseline de rendimiento del sistema con carga mínima.

**Instrucciones:**

1. Crear el archivo `tests/load/smoke-test.js`:

```javascript
// tests/load/smoke-test.js
// Smoke test: 1 VU, 1 minuto - verificar que el sistema responde correctamente
import http from 'k6/http';
import { check, sleep } from 'k6';
import { Rate, Trend } from 'k6/metrics';

// Métricas personalizadas
const errorRate = new Rate('errors');
const orderLatency = new Trend('order_service_latency', true);
const inventoryLatency = new Trend('inventory_service_latency', true);

export const options = {
  vus: 1,
  duration: '1m',
  thresholds: {
    http_req_duration: ['p(95)<300'],
    errors: ['rate<0.01'],
  },
  tags: {
    testType: 'smoke',
  },
};

const BASE_URL_ORDER = __ENV.ORDER_URL || 'http://localhost:8001';
const BASE_URL_INVENTORY = __ENV.INVENTORY_URL || 'http://localhost:8002';

export default function () {
  // Test order-service health
  let orderHealth = http.get(`${BASE_URL_ORDER}/health`);
  check(orderHealth, {
    'order-service status 200': (r) => r.status === 200,
    'order-service healthy': (r) => r.json().status === 'healthy',
  }) || errorRate.add(1);
  orderLatency.add(orderHealth.timings.duration);

  // Test inventory-service health
  let invHealth = http.get(`${BASE_URL_INVENTORY}/health`);
  check(invHealth, {
    'inventory-service status 200': (r) => r.status === 200,
    'inventory-service healthy': (r) => r.json().status === 'healthy',
  }) || errorRate.add(1);
  inventoryLatency.add(invHealth.timings.duration);

  // Test order creation endpoint
  let orderPayload = JSON.stringify({
    product_id: 'prod-001',
    quantity: 1,
    customer_id: 'cust-smoke-test',
  });

  let orderRes = http.post(`${BASE_URL_ORDER}/api/orders`, orderPayload, {
    headers: { 'Content-Type': 'application/json' },
  });

  check(orderRes, {
    'order created status 201 or 200': (r) => r.status === 201 || r.status === 200,
    'order has id': (r) => r.json().id !== undefined || r.json().order_id !== undefined,
  }) || errorRate.add(1);

  // Test inventory query
  let invRes = http.get(`${BASE_URL_INVENTORY}/api/inventory/prod-001`);
  check(invRes, {
    'inventory query status 200': (r) => r.status === 200,
  }) || errorRate.add(1);

  sleep(1);
}

export function handleSummary(data) {
  return {
    'tests/load/results/smoke-test-summary.json': JSON.stringify(data, null, 2),
  };
}
```

2. Crear directorio de resultados y ejecutar:

```bash
mkdir -p tests/load/results
k6 run tests/load/smoke-test.js
```

**Salida esperada:**

```
     ✓ order-service status 200
     ✓ order-service healthy
     ✓ inventory-service status 200
     ✓ inventory-service healthy

     checks.........................: 100.00% ✓ XX  ✗ 0
     http_req_duration..............: avg=XXms  min=XXms  max=XXms  p(95)=XXms
     errors.........................: 0.00%   ✓ 0   ✗ XX
```

**Verificación:** Todos los checks pasan al 100%, p(95) < 300ms, error rate = 0%.

---

### Paso 3: Crear Script de Load Test con k6

**Objetivo:** Implementar un load test con rampas progresivas que identifique el comportamiento del sistema bajo carga sostenida.

**Instrucciones:**

1. Crear el archivo `tests/load/load-test.js`:

```javascript
// tests/load/load-test.js
// Load test: rampa 0→50 VUs en 5min, sostenido 10min, bajada 5min
import http from 'k6/http';
import { check, sleep } from 'k6';
import { Rate, Trend, Counter } from 'k6/metrics';

const errorRate = new Rate('errors');
const orderCreated = new Counter('orders_created');
const orderLatency = new Trend('order_latency_ms', true);

export const options = {
  stages: [
    { duration: '5m', target: 50 },   // Rampa de subida
    { duration: '10m', target: 50 },   // Carga sostenida
    { duration: '5m', target: 0 },     // Rampa de bajada
  ],
  thresholds: {
    http_req_duration: ['p(95)<500', 'p(99)<1000'],
    errors: ['rate<0.01'],
    http_req_failed: ['rate<0.01'],
  },
  tags: {
    testType: 'load',
  },
};

const BASE_URL_ORDER = __ENV.ORDER_URL || 'http://localhost:8001';
const BASE_URL_INVENTORY = __ENV.INVENTORY_URL || 'http://localhost:8002';

// Datos de prueba variados
const products = ['prod-001', 'prod-002', 'prod-003', 'prod-004', 'prod-005'];
const quantities = [1, 2, 3, 5, 10];

function getRandomElement(arr) {
  return arr[Math.floor(Math.random() * arr.length)];
}

export default function () {
  const productId = getRandomElement(products);
  const quantity = getRandomElement(quantities);

  // 60% lecturas, 40% escrituras (patrón realista)
  if (Math.random() < 0.6) {
    // Consultar inventario
    let invRes = http.get(`${BASE_URL_INVENTORY}/api/inventory/${productId}`);
    check(invRes, {
      'inventory GET 200': (r) => r.status === 200,
      'inventory response time OK': (r) => r.timings.duration < 500,
    }) || errorRate.add(1);
  } else {
    // Crear orden
    let payload = JSON.stringify({
      product_id: productId,
      quantity: quantity,
      customer_id: `cust-load-${__VU}-${__ITER}`,
    });

    let orderRes = http.post(`${BASE_URL_ORDER}/api/orders`, payload, {
      headers: { 'Content-Type': 'application/json' },
    });

    let success = check(orderRes, {
      'order POST success': (r) => r.status === 200 || r.status === 201,
      'order response time OK': (r) => r.timings.duration < 500,
    });

    if (success) {
      orderCreated.add(1);
    } else {
      errorRate.add(1);
    }
    orderLatency.add(orderRes.timings.duration);
  }

  sleep(Math.random() * 2 + 0.5); // 0.5-2.5s entre requests
}

export function handleSummary(data) {
  return {
    'tests/load/results/load-test-summary.json': JSON.stringify(data, null, 2),
    stdout: textSummary(data, { indent: '  ', enableColors: true }),
  };
}

import { textSummary } from 'https://jslib.k6.io/k6-summary/0.0.2/index.js';
```

2. Ejecutar el load test (versión reducida para el laboratorio — 2 min total):

```bash
# Versión reducida para el contexto del lab (ajustar stages si se desea full)
k6 run --duration 2m --vus 20 tests/load/load-test.js
```

**Salida esperada:**

```
     ✓ inventory GET 200
     ✓ order POST success
     ✓ order response time OK

     checks.........................: 9X.XX% ✓ XXXX  ✗ X
     http_req_duration..............: avg=XXms   p(95)=XXXms  p(99)=XXXms
     errors.........................: X.XX%
     orders_created.................: XXXX
```

**Verificación:** p(95) < 500ms, error rate < 1%, throughput sostenido visible en el resumen.

---

### Paso 4: Crear Scripts de Stress Test y Spike Test

**Objetivo:** Implementar escenarios extremos para identificar el punto de quiebre del sistema y su comportamiento ante picos súbitos.

**Instrucciones:**

1. Crear `tests/load/stress-test.js`:

```javascript
// tests/load/stress-test.js
// Stress test: incremento escalonado hasta encontrar punto de quiebre
import http from 'k6/http';
import { check, sleep } from 'k6';
import { Rate } from 'k6/metrics';

const errorRate = new Rate('errors');

export const options = {
  stages: [
    { duration: '2m', target: 50 },    // Nivel normal
    { duration: '2m', target: 100 },   // Nivel elevado
    { duration: '2m', target: 150 },   // Nivel alto
    { duration: '2m', target: 200 },   // Nivel crítico
    { duration: '2m', target: 250 },   // Punto de quiebre esperado
    { duration: '2m', target: 0 },     // Recuperación
  ],
  thresholds: {
    http_req_duration: ['p(95)<2000'],  // Umbral relajado para stress
    errors: ['rate<0.15'],              // Hasta 15% errores aceptable en stress
  },
  tags: {
    testType: 'stress',
  },
};

const BASE_URL_ORDER = __ENV.ORDER_URL || 'http://localhost:8001';

export default function () {
  let payload = JSON.stringify({
    product_id: `prod-${String(Math.floor(Math.random() * 100)).padStart(3, '0')}`,
    quantity: Math.floor(Math.random() * 10) + 1,
    customer_id: `stress-${__VU}-${__ITER}`,
  });

  let res = http.post(`${BASE_URL_ORDER}/api/orders`, payload, {
    headers: { 'Content-Type': 'application/json' },
    timeout: '10s',
  });

  check(res, {
    'status is not 5xx': (r) => r.status < 500,
    'response within 2s': (r) => r.timings.duration < 2000,
  }) || errorRate.add(1);

  sleep(0.3);
}

export function handleSummary(data) {
  return {
    'tests/load/results/stress-test-summary.json': JSON.stringify(data, null, 2),
  };
}
```

2. Crear `tests/load/spike-test.js`:

```javascript
// tests/load/spike-test.js
// Spike test: pico súbito de 0 a 200 VUs en 10 segundos
import http from 'k6/http';
import { check, sleep } from 'k6';
import { Rate, Trend } from 'k6/metrics';

const errorRate = new Rate('errors');
const spikeLatency = new Trend('spike_latency', true);

export const options = {
  stages: [
    { duration: '30s', target: 5 },     // Warm-up mínimo
    { duration: '10s', target: 200 },    // SPIKE: subida abrupta
    { duration: '1m', target: 200 },     // Mantener pico
    { duration: '10s', target: 5 },      // Bajada abrupta
    { duration: '1m', target: 5 },       // Recuperación
  ],
  thresholds: {
    http_req_duration: ['p(99)<3000'],
    errors: ['rate<0.20'],  // Hasta 20% errores durante spike
  },
  tags: {
    testType: 'spike',
  },
};

const BASE_URL_ORDER = __ENV.ORDER_URL || 'http://localhost:8001';
const BASE_URL_INVENTORY = __ENV.INVENTORY_URL || 'http://localhost:8002';

export default function () {
  let res = http.get(`${BASE_URL_ORDER}/health`);
  check(res, {
    'still responding': (r) => r.status === 200,
  }) || errorRate.add(1);
  spikeLatency.add(res.timings.duration);

  let invRes = http.get(`${BASE_URL_INVENTORY}/api/inventory/prod-001`);
  check(invRes, {
    'inventory available during spike': (r) => r.status === 200,
  }) || errorRate.add(1);

  sleep(0.1);
}

export function handleSummary(data) {
  return {
    'tests/load/results/spike-test-summary.json': JSON.stringify(data, null, 2),
  };
}
```

3. Ejecutar el spike test (versión reducida):

```bash
k6 run tests/load/spike-test.js
```

**Salida esperada:**

```
     ✗ still responding
      ↳  9X% — ✓ XXXX / ✗ XX

     http_req_duration..............: avg=XXXms  p(95)=XXXXms  p(99)=XXXXms
     errors.........................: X.XX%
```

**Verificación:** El sistema muestra degradación durante el spike pero se recupera en la fase de bajada. Documentar el p99 máximo observado y el porcentaje de errores durante el pico.

---

### Paso 5: Configurar Exportación de Métricas k6 a Prometheus

**Objetivo:** Habilitar la exportación de métricas de k6 a Prometheus para visualización unificada en Grafana.

**Instrucciones:**

1. Verificar que Prometheus tiene habilitado el remote write receiver:

```bash
# Verificar la configuración de Prometheus
kubectl get prometheus -n monitoring -o yaml | grep -A5 "remoteWrite"
```

2. Si no está habilitado, parchear la configuración:

```bash
# Habilitar remote write receiver en Prometheus
kubectl patch prometheus prometheus-kube-prometheus-prometheus -n monitoring \
  --type merge -p '{"spec":{"enableRemoteWriteReceiver":true}}'
```

3. Ejecutar k6 con output a Prometheus remote write:

```bash
# Obtener la URL del servicio Prometheus
PROM_URL="http://localhost:9090/api/v1/write"

# Ejecutar smoke test con exportación a Prometheus
k6 run --out experimental-prometheus-rw \
  -e K6_PROMETHEUS_RW_SERVER_URL=$PROM_URL \
  -e K6_PROMETHEUS_RW_TREND_AS_NATIVE_HISTOGRAM=true \
  tests/load/smoke-test.js
```

4. Verificar que las métricas llegaron a Prometheus:

```bash
# Consultar métricas k6 en Prometheus
curl -s "http://localhost:9090/api/v1/query?query=k6_http_req_duration_p95" | jq '.data.result[0].value'
```

**Salida esperada:** Un valor numérico representando la latencia p95 en milisegundos.

**Verificación:** Las métricas `k6_*` aparecen en Prometheus y pueden consultarse via PromQL.

---

### Paso 6: Implementar Experimento PodChaos — Kill de Pods

**Objetivo:** Validar que el sistema se recupera automáticamente cuando se eliminan pods de order-service.

**Instrucciones:**

1. Crear el manifiesto `tests/chaos/pod-kill-order.yaml`:

```yaml
# tests/chaos/pod-kill-order.yaml
apiVersion: chaos-mesh.org/v1alpha1
kind: PodChaos
metadata:
  name: pod-kill-order-service
  namespace: chaos-testing
  labels:
    experiment: pod-kill
    target: order-service
spec:
  action: pod-kill
  mode: one
  selector:
    namespaces:
      - microservices-prod
    labelSelectors:
      app: order-service
  scheduler:
    cron: "*/30 * * * * *"  # Cada 30 segundos
  duration: "3m"
```

2. Iniciar el load test en una terminal separada:

```bash
# Terminal 1: ejecutar load test durante el experimento
k6 run --duration 4m --vus 10 tests/load/load-test.js 2>&1 | tee tests/chaos/results/pod-kill-loadtest.log &
LOAD_TEST_PID=$!
```

3. Aplicar el experimento de caos:

```bash
# Terminal 2: aplicar el experimento
mkdir -p tests/chaos/results
kubectl apply -f tests/chaos/pod-kill-order.yaml

# Observar los pods siendo eliminados y recreados
watch -n 5 'kubectl get pods -n microservices-prod -l app=order-service'
```

4. Monitorear la recuperación:

```bash
# Ver eventos del namespace
kubectl get events -n microservices-prod --sort-by='.lastTimestamp' | tail -20

# Verificar que el Deployment mantiene los replicas deseados
kubectl get deployment order-service -n microservices-prod
```

5. Limpiar el experimento después de 3 minutos:

```bash
kubectl delete -f tests/chaos/pod-kill-order.yaml
wait $LOAD_TEST_PID
```

**Salida esperada:**

```
podchaos.chaos-mesh.org/pod-kill-order-service created
NAME                             READY   STATUS    RESTARTS   AGE
order-service-xxxxx-yyyyy        1/1     Running   0          5s
```

**Verificación:** Los pods se recrean en < 30 segundos. El load test muestra errores transitorios (< 5%) durante los kills pero se recupera. Documentar el tiempo de recuperación promedio.

---

### Paso 7: Implementar Experimento NetworkChaos — Inyección de Latencia

**Objetivo:** Medir el impacto en SLOs cuando se introduce latencia de 200ms en la comunicación entre servicios.

**Instrucciones:**

1. Crear `tests/chaos/network-delay-services.yaml`:

```yaml
# tests/chaos/network-delay-services.yaml
apiVersion: chaos-mesh.org/v1alpha1
kind: NetworkChaos
metadata:
  name: network-delay-order-to-inventory
  namespace: chaos-testing
  labels:
    experiment: network-delay
    target: inter-service
spec:
  action: delay
  mode: all
  selector:
    namespaces:
      - microservices-prod
    labelSelectors:
      app: order-service
  delay:
    latency: "200ms"
    correlation: "80"
    jitter: "50ms"
  direction: to
  target:
    selector:
      namespaces:
        - microservices-prod
      labelSelectors:
        app: inventory-service
    mode: all
  duration: "5m"
```

2. Ejecutar load test y aplicar el experimento simultáneamente:

```bash
# Iniciar load test
k6 run --duration 6m --vus 15 tests/load/load-test.js \
  2>&1 | tee tests/chaos/results/network-delay-loadtest.log &
LOAD_PID=$!

# Esperar 30s para establecer baseline, luego inyectar latencia
sleep 30
kubectl apply -f tests/chaos/network-delay-services.yaml
echo "$(date): NetworkChaos delay aplicado"

# Esperar a que termine
wait $LOAD_PID
```

3. Consultar el impacto en Prometheus:

```bash
# Latencia p95 durante el experimento
curl -s "http://localhost:9090/api/v1/query?query=histogram_quantile(0.95,rate(http_request_duration_seconds_bucket{service=\"order-service\"}[1m]))" | jq .

# Tasa de errores
curl -s "http://localhost:9090/api/v1/query?query=rate(http_requests_total{service=\"order-service\",status=~\"5..\"}[1m])/rate(http_requests_total{service=\"order-service\"}[1m])" | jq .
```

4. Limpiar:

```bash
kubectl delete -f tests/chaos/network-delay-services.yaml
```

**Salida esperada:** La latencia p95 sube de ~100ms a ~350-400ms durante el experimento. Los errores se mantienen bajos si los timeouts están configurados correctamente.

**Verificación:** Comparar las métricas antes y durante el experimento. El incremento de latencia debe ser aproximadamente +200ms ± 50ms (el jitter configurado).

---

### Paso 8: Implementar Experimento NetworkChaos — Partición de Red

**Objetivo:** Simular una partición de red entre inventory-service y PostgreSQL para validar el manejo de errores de base de datos.

**Instrucciones:**

1. Crear `tests/chaos/network-partition-db.yaml`:

```yaml
# tests/chaos/network-partition-db.yaml
apiVersion: chaos-mesh.org/v1alpha1
kind: NetworkChaos
metadata:
  name: partition-inventory-postgres
  namespace: chaos-testing
  labels:
    experiment: network-partition
    target: database
spec:
  action: partition
  mode: all
  selector:
    namespaces:
      - microservices-prod
    labelSelectors:
      app: inventory-service
  direction: both
  target:
    selector:
      namespaces:
        - microservices-prod
      labelSelectors:
        app: postgres
    mode: all
  duration: "2m"
```

2. Ejecutar y observar:

```bash
# Iniciar load test con foco en inventory
k6 run --duration 3m --vus 10 tests/load/load-test.js \
  2>&1 | tee tests/chaos/results/partition-db-loadtest.log &
LOAD_PID=$!

# Esperar baseline y aplicar partición
sleep 30
echo "$(date '+%Y-%m-%d %H:%M:%S'): Aplicando partición de red DB" >> tests/chaos/results/timeline.log
kubectl apply -f tests/chaos/network-partition-db.yaml

# Monitorear errores del inventory-service
kubectl logs -n microservices-prod -l app=inventory-service --tail=20 -f &
LOG_PID=$!

# Esperar fin del experimento
sleep 150
kubectl delete -f tests/chaos/network-partition-db.yaml
echo "$(date '+%Y-%m-%d %H:%M:%S'): Partición de red removida" >> tests/chaos/results/timeline.log

kill $LOG_PID 2>/dev/null
wait $LOAD_PID
```

**Salida esperada:** inventory-service retorna errores 503 durante la partición. Los logs muestran errores de conexión a PostgreSQL. Después de remover la partición, el servicio se recupera automáticamente.

**Verificación:** Registrar el porcentaje de errores durante la partición (esperado: ~100% para operaciones de inventory que requieren DB) y el tiempo de recuperación post-partición (esperado: < 30 segundos).

---

### Paso 9: Implementar Experimento StressChaos — CPU Stress

**Objetivo:** Evaluar el comportamiento del sistema cuando order-service experimenta contención de CPU al 80%.

**Instrucciones:**

1. Crear `tests/chaos/cpu-stress-order.yaml`:

```yaml
# tests/chaos/cpu-stress-order.yaml
apiVersion: chaos-mesh.org/v1alpha1
kind: StressChaos
metadata:
  name: cpu-stress-order-service
  namespace: chaos-testing
  labels:
    experiment: cpu-stress
    target: order-service
spec:
  mode: all
  selector:
    namespaces:
      - microservices-prod
    labelSelectors:
      app: order-service
  stressors:
    cpu:
      workers: 2
      load: 80
  duration: "3m"
```

2. Ejecutar con load test paralelo:

```bash
# Load test durante CPU stress
k6 run --duration 4m --vus 15 tests/load/load-test.js \
  2>&1 | tee tests/chaos/results/cpu-stress-loadtest.log &
LOAD_PID=$!

sleep 30
echo "$(date '+%Y-%m-%d %H:%M:%S'): Aplicando CPU stress 80%" >> tests/chaos/results/timeline.log
kubectl apply -f tests/chaos/cpu-stress-order.yaml

# Monitorear uso de CPU
kubectl top pods -n microservices-prod -l app=order-service

wait $LOAD_PID
kubectl delete -f tests/chaos/cpu-stress-order.yaml
echo "$(date '+%Y-%m-%d %H:%M:%S'): CPU stress removido" >> tests/chaos/results/timeline.log
```

**Salida esperada:** La latencia p95 aumenta significativamente (2-5x) durante el stress. El throughput puede disminuir pero los errores deben mantenerse bajos si no hay timeouts agresivos.

**Verificación:** Documentar la degradación de latencia y throughput. El sistema no debe crashear — solo degradarse gracefully.

---

### Paso 10: Ejecutar Contract Testing Durante Caos

**Objetivo:** Validar que los contratos de API se mantienen íntegros incluso durante condiciones de estrés.

**Instrucciones:**

1. Instalar schemathesis si no está disponible:

```bash
pip install schemathesis==3.27.1
```

2. Ejecutar contract testing contra ambos servicios en estado normal:

```bash
# Contract test contra order-service
schemathesis run http://localhost:8001/openapi.json \
  --checks all \
  --hypothesis-max-examples=50 \
  --request-timeout=5000 \
  --validate-schema=true \
  2>&1 | tee tests/contracts/order-service-baseline.log

# Contract test contra inventory-service
schemathesis run http://localhost:8002/openapi.json \
  --checks all \
  --hypothesis-max-examples=50 \
  --request-timeout=5000 \
  --validate-schema=true \
  2>&1 | tee tests/contracts/inventory-service-baseline.log
```

3. Ejecutar contract testing DURANTE un experimento de caos:

```bash
# Aplicar network delay
kubectl apply -f tests/chaos/network-delay-services.yaml

# Esperar 15s para que el caos se estabilice
sleep 15

# Contract test durante caos
schemathesis run http://localhost:8001/openapi.json \
  --checks all \
  --hypothesis-max-examples=30 \
  --request-timeout=10000 \
  --validate-schema=true \
  2>&1 | tee tests/contracts/order-service-under-chaos.log

# Limpiar
kubectl delete -f tests/chaos/network-delay-services.yaml
```

4. Comparar resultados:

```bash
echo "=== Baseline ==="
grep -E "(PASSED|FAILED|ERROR)" tests/contracts/order-service-baseline.log
echo ""
echo "=== Under Chaos ==="
grep -E "(PASSED|FAILED|ERROR)" tests/contracts/order-service-under-chaos.log
```

**Salida esperada:** Los contratos de API (estructura de respuesta, tipos de datos, códigos de estado documentados) deben pasar incluso bajo caos. Los errores de timeout son aceptables pero las respuestas que sí llegan deben cumplir el esquema OpenAPI.

**Verificación:** No hay violaciones de contrato (schema mismatches). Los errores son solo de disponibilidad (timeouts, 503s) no de formato.

---

### Paso 11: Definir SLOs y Configurar Alertas

**Objetivo:** Establecer SLOs formales con alertas en Alertmanager que notifiquen cuando el error budget se consume excesivamente.

**Instrucciones:**

1. Crear `monitoring/alerts/slo-rules.yaml`:

```yaml
# monitoring/alerts/slo-rules.yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: microservices-slo-rules
  namespace: monitoring
  labels:
    release: prometheus
spec:
  groups:
    - name: slo.availability
      interval: 30s
      rules:
        # Recording rule: tasa de éxito de order-service
        - record: slo:order_service:availability:rate5m
          expr: |
            sum(rate(http_requests_total{service="order-service",status!~"5.."}[5m]))
            /
            sum(rate(http_requests_total{service="order-service"}[5m]))

        # Recording rule: tasa de éxito de inventory-service
        - record: slo:inventory_service:availability:rate5m
          expr: |
            sum(rate(http_requests_total{service="inventory-service",status!~"5.."}[5m]))
            /
            sum(rate(http_requests_total{service="inventory-service"}[5m]))

        # Recording rule: latencia p95 de order-service
        - record: slo:order_service:latency_p95:5m
          expr: |
            histogram_quantile(0.95,
              sum(rate(http_request_duration_seconds_bucket{service="order-service"}[5m])) by (le)
            )

    - name: slo.alerts
      rules:
        # Alerta: disponibilidad por debajo del SLO (99.5%)
        - alert: OrderServiceAvailabilityBelowSLO
          expr: slo:order_service:availability:rate5m < 0.995
          for: 2m
          labels:
            severity: critical
            team: platform
          annotations:
            summary: "order-service availability below 99.5% SLO"
            description: |
              Current availability: {{ $value | humanizePercentage }}.
              SLO target: 99.5%. Error budget being consumed rapidly.
            runbook_url: "https://docs.internal/runbooks/order-service-availability"

        # Alerta: latencia por encima del SLO (p95 < 500ms)
        - alert: OrderServiceLatencyAboveSLO
          expr: slo:order_service:latency_p95:5m > 0.5
          for: 3m
          labels:
            severity: warning
            team: platform
          annotations:
            summary: "order-service p95 latency above 500ms SLO"
            description: |
              Current p95 latency: {{ $value | humanizeDuration }}.
              SLO target: <500ms.

        # Alerta: error budget consumido >50% en 1 hora
        - alert: ErrorBudgetBurnRateHigh
          expr: |
            (1 - slo:order_service:availability:rate5m) / (1 - 0.995) > 0.5
          for: 5m
          labels:
            severity: critical
            team: platform
          annotations:
            summary: "Error budget burn rate exceeds 50%"
            description: |
              Error budget is being consumed at {{ $value | humanizePercentage }}
              of the monthly allowance per hour. Immediate investigation required.

        # Alerta: inventory-service no disponible
        - alert: InventoryServiceUnavailable
          expr: |
            sum(rate(http_requests_total{service="inventory-service",status=~"5.."}[2m]))
            /
            sum(rate(http_requests_total{service="inventory-service"}[2m])) > 0.5
          for: 1m
          labels:
            severity: critical
            team: platform
          annotations:
            summary: "inventory-service error rate above 50%"
            runbook_url: "https://docs.internal/runbooks/inventory-service-unavailable"
```

2. Aplicar las reglas:

```bash
kubectl apply -f monitoring/alerts/slo-rules.yaml
```

3. Verificar que las reglas se cargaron:

```bash
# Verificar en Prometheus
curl -s http://localhost:9090/api/v1/rules | jq '.data.groups[] | select(.name | startswith("slo")) | .rules[].name'
```

**Salida esperada:**

```
"slo:order_service:availability:rate5m"
"slo:inventory_service:availability:rate5m"
"slo:order_service:latency_p95:5m"
"OrderServiceAvailabilityBelowSLO"
"OrderServiceLatencyAboveSLO"
"ErrorBudgetBurnRateHigh"
"InventoryServiceUnavailable"
```

**Verificación:** Las recording rules producen valores y las alertas están en estado `inactive` (no disparadas) cuando el sistema está sano.

---

### Paso 12: Crear Runbook Operacional

**Objetivo:** Documentar un runbook para el escenario "inventory-service-unavailable" con pasos de diagnóstico, mitigación y escalación.

**Instrucciones:**

1. Crear `docs/runbooks/inventory-service-unavailable.md`:

```markdown
# Runbook: inventory-service-unavailable

## Metadata
- **Alerta:** InventoryServiceUnavailable
- **Severidad:** Critical
- **Equipo responsable:** Platform Engineering
- **Última actualización:** $(date +%Y-%m-%d)
- **SLO afectado:** Disponibilidad ≥99.5%

## Descripción
Esta alerta se dispara cuando la tasa de errores 5xx del inventory-service
supera el 50% durante más de 1 minuto. Indica una degradación severa que
impacta directamente la capacidad de crear órdenes.

## Impacto
- **Usuarios afectados:** Todos los que intentan crear órdenes (order-service depende de inventory-service)
- **Funcionalidad degradada:** Consulta de stock, reserva de inventario
- **Cascada potencial:** order-service puede acumular timeouts y degradarse

## Diagnóstico

### Paso 1: Verificar estado de pods
```bash
kubectl get pods -n microservices-prod -l app=inventory-service
kubectl describe pod -n microservices-prod -l app=inventory-service
```

### Paso 2: Revisar logs recientes
```bash
kubectl logs -n microservices-prod -l app=inventory-service --tail=100 --since=5m
```

### Paso 3: Verificar conectividad a PostgreSQL
```bash
kubectl exec -n microservices-prod deploy/inventory-service -- \
  python -c "import asyncpg; import asyncio; asyncio.run(asyncpg.connect('postgresql://mslab_user:mslab_pass_2024@postgres:5432/inventory_db'))"
```

### Paso 4: Verificar métricas en Prometheus
```promql
# Tasa de errores actual
rate(http_requests_total{service="inventory-service",status=~"5.."}[2m])

# Latencia actual
histogram_quantile(0.95, rate(http_request_duration_seconds_bucket{service="inventory-service"}[2m]))

# Conexiones a DB
pg_stat_activity_count{datname="inventory_db"}
```

### Paso 5: Verificar recursos del pod
```bash
kubectl top pods -n microservices-prod -l app=inventory-service
```

## Mitigación

### Opción A: Restart de pods (si hay memory leak o estado corrupto)
```bash
kubectl rollout restart deployment/inventory-service -n microservices-prod
kubectl rollout status deployment/inventory-service -n microservices-prod --timeout=60s
```

### Opción B: Escalar horizontalmente (si hay saturación)
```bash
kubectl scale deployment/inventory-service -n microservices-prod --replicas=5
```

### Opción C: Verificar y reiniciar PostgreSQL (si DB no responde)
```bash
kubectl get pods -n microservices-prod -l app=postgres
kubectl delete pod -n microservices-prod -l app=postgres  # StatefulSet lo recrea
```

## Escalación
| Tiempo sin resolver | Acción |
|---|---|
| 5 min | Notificar al Tech Lead de turno |
| 15 min | Escalar a Engineering Manager |
| 30 min | Activar incident commander, comunicar a stakeholders |

## Resolución
- Confirmar que la tasa de errores vuelve a <1%
- Verificar que la alerta se resuelve automáticamente
- Crear ticket de postmortem si el incidente duró >5 min
```

2. Guardar el archivo:

```bash
cat docs/runbooks/inventory-service-unavailable.md | head -5
```

**Verificación:** El runbook contiene secciones de diagnóstico, mitigación y escalación con comandos ejecutables.

---

### Paso 13: Redactar Postmortem Estructurado

**Objetivo:** Documentar el experimento de NetworkChaos partition como un postmortem de incidente real siguiendo la plantilla Google SRE.

**Instrucciones:**

1. Crear `docs/postmortems/2024-XX-XX-network-partition-inventory-db.md` (reemplazar XX con fecha actual):

```bash
cat > docs/postmortems/$(date +%Y-%m-%d)-network-partition-inventory-db.md << 'POSTMORTEM'
# Postmortem: Partición de Red entre inventory-service y PostgreSQL

## Resumen Ejecutivo

| Campo | Valor |
|-------|-------|
| **Fecha del incidente** | $(date +%Y-%m-%d) |
| **Duración** | 2 minutos 15 segundos |
| **Severidad** | SEV-2 |
| **Servicios afectados** | inventory-service, order-service (cascada) |
| **Impacto en usuarios** | ~100% de requests a inventory fallaron; ~40% de creación de órdenes fallaron |
| **SLOs violados** | Disponibilidad (cayó a ~0% durante partición), Latencia (timeouts) |
| **Autor** | [Tu nombre] |
| **Revisores** | [Equipo de plataforma] |

## Impacto

- **Requests fallidos:** ~XXX requests retornaron 503 durante la ventana de 2 minutos
- **Error budget consumido:** Aproximadamente 0.14% del budget mensual (30d)
- **Usuarios afectados:** Todos los usuarios intentando consultar inventario o crear órdenes
- **Revenue impact:** Estimado en $0 (entorno de laboratorio)

## Timeline (UTC)

| Hora | Evento |
|------|--------|
| HH:MM:00 | Load test iniciado con 10 VUs |
| HH:MM:30 | Baseline establecido: p95=80ms, error_rate=0% |
| HH:MM:30 | **INICIO:** NetworkChaos partition aplicada |
| HH:MM:35 | Primeros errores 503 en inventory-service |
| HH:MM:40 | order-service comienza a retornar errores por timeout a inventory |
| HH:MM:45 | Alerta InventoryServiceUnavailable disparada |
| HH:MM:00 | Alerta OrderServiceAvailabilityBelowSLO disparada |
| HH:+2:30 | **FIN:** NetworkChaos partition removida |
| HH:+2:35 | inventory-service restablece conexión a PostgreSQL |
| HH:+2:45 | Pool de conexiones completamente recuperado |
| HH:+3:00 | Métricas vuelven a niveles normales |
| HH:+3:00 | Alertas se resuelven automáticamente |

## Root Cause Analysis

### Causa raíz
La partición de red entre inventory-service y PostgreSQL causó que todas las
queries a la base de datos fallaran con timeout. El pool de conexiones de
asyncpg agotó sus reintentos y comenzó a retornar errores inmediatamente.

### Factores contribuyentes
1. **Sin circuit breaker a nivel de DB:** Las conexiones fallidas no se cortaron
   rápidamente, causando acumulación de requests en espera
2. **Sin fallback/cache:** inventory-service no tiene cache de lectura que
   pudiera servir datos stale durante la partición
3. **Acoplamiento síncrono:** order-service depende síncronamente de
   inventory-service sin degradación graceful
4. **Health check no detectó la partición:** El endpoint /health no verifica
   conectividad a DB

## Lecciones Aprendidas

### Lo que funcionó bien
- El Deployment controller de Kubernetes mantuvo los pods running
- Las alertas se dispararon dentro del tiempo configurado (< 2 min)
- La recuperación fue automática una vez restaurada la red
- Los logs proporcionaron información clara del error

### Lo que no funcionó
- No hay circuit breaker para conexiones a PostgreSQL
- No hay cache de lectura para degradación graceful
- El health check no refleja el estado real de dependencias
- order-service no tiene fallback cuando inventory no responde

## Action Items

| # | Acción | Prioridad | Owner | Fecha límite | Estado |
|---|--------|-----------|-------|--------------|--------|
| 1 | Implementar circuit breaker (pybreaker) para conexiones DB en inventory-service | P1 | [Dev Lead] | +2 semanas | TODO |
| 2 | Agregar Redis cache de lectura para consultas de inventario frecuentes | P1 | [Backend Dev] | +3 semanas | TODO |
| 3 | Implementar health check con verificación de dependencias (/health/ready) | P2 | [Platform Eng] | +1 semana | TODO |
| 4 | Agregar fallback en order-service: aceptar orden con validación async si inventory no responde | P2 | [Dev Lead] | +4 semanas | TODO |
| 5 | Configurar PodDisruptionBudget para inventory-service | P3 | [Platform Eng] | +2 semanas | TODO |
| 6 | Agregar dashboard de conectividad DB en Grafana | P3 | [SRE] | +1 semana | TODO |

## Métricas del Incidente

```promql
# Disponibilidad durante el incidente (ventana de 5 min alrededor del evento)
avg_over_time(slo:inventory_service:availability:rate5m[5m])

# Error budget consumido
1 - avg_over_time(slo:inventory_service:availability:rate5m[2m])
```

## Prevención Futura

Para evitar que este escenario cause impacto en producción:
1. Circuit breaker cortará conexiones fallidas en <5 requests
2. Cache permitirá servir datos stale (max 30s) durante particiones
3. Health check /ready fallará, removiendo el pod del Service y evitando tráfico
4. order-service aceptará órdenes condicionalmente, validando inventario async
POSTMORTEM
```

2. Verificar el postmortem:

```bash
wc -l docs/postmortems/$(date +%Y-%m-%d)-network-partition-inventory-db.md
head -20 docs/postmortems/$(date +%Y-%m-%d)-network-partition-inventory-db.md
```

**Salida esperada:** El archivo contiene ~120+ líneas con todas las secciones del postmortem.

**Verificación:** El postmortem incluye: resumen ejecutivo, impacto cuantificado, timeline con timestamps, root cause, lecciones aprendidas, y action items con owners y fechas.

---

### Paso 14: Validar Recuperación de SLOs Post-Experimentos

**Objetivo:** Confirmar que el sistema recupera sus SLOs después de todos los experimentos de caos y que el pipeline CI/CD funciona durante caos activo.

**Instrucciones:**

1. Verificar que no hay experimentos de caos activos:

```bash
kubectl get podchaos,networkchaos,stresschaos -n chaos-testing
```

Debe retornar `No resources found`.

2. Ejecutar smoke test final para confirmar estado saludable:

```bash
k6 run tests/load/smoke-test.js 2>&1 | tee tests/load/results/final-validation.log
```

3. Verificar SLOs en Prometheus:

```bash
# Disponibilidad actual (debe ser ~1.0)
curl -s "http://localhost:9090/api/v1/query?query=slo:order_service:availability:rate5m" | \
  jq '.data.result[0].value[1]'

# Latencia p95 actual (debe ser <0.5)
curl -s "http://localhost:9090/api/v1/query?query=slo:order_service:latency_p95:5m" | \
  jq '.data.result[0].value[1]'
```

4. Verificar que no hay alertas activas:

```bash
curl -s http://localhost:9090/api/v1/alerts | jq '.data.alerts[] | select(.state=="firing")'
```

5. Verificar que el pipeline CI/CD funciona (simular trigger):

```bash
# Verificar que Argo CD puede sincronizar
kubectl get application -n argocd -o jsonpath='{range .items[*]}{.metadata.name}: {.status.sync.status} / {.status.health.status}{"\n"}{end}'
```

**Salida esperada:**

```
order-service: Synced / Healthy
inventory-service: Synced / Healthy
```

**Verificación:** 
- Smoke test: 100% checks passing
- Disponibilidad: > 0.995
- Latencia p95: < 0.5s
- Alertas activas: ninguna
- Argo CD: Synced/Healthy

---

## Validación y Pruebas Finales

Ejecutar la siguiente lista de verificación para confirmar la completitud del laboratorio:

```bash
echo "=== VALIDACIÓN FINAL DEL LABORATORIO ==="
echo ""

# 1. Archivos de pruebas de carga
echo "1. Scripts k6 creados:"
ls -la tests/load/*.js | wc -l
echo "   Esperado: 4 archivos (smoke, load, stress, spike)"

# 2. Resultados de pruebas
echo ""
echo "2. Resultados de pruebas:"
ls tests/load/results/ 2>/dev/null | wc -l
echo "   Esperado: ≥4 archivos de resultados"

# 3. Manifiestos de Chaos Mesh
echo ""
echo "3. Manifiestos de caos:"
ls tests/chaos/*.yaml | wc -l
echo "   Esperado: 4 archivos (pod-kill, network-delay, partition, cpu-stress)"

# 4. Resultados de caos
echo ""
echo "4. Logs de experimentos de caos:"
ls tests/chaos/results/ 2>/dev/null | wc -l

# 5. Contract testing
echo ""
echo "5. Resultados de contract testing:"
ls tests/contracts/*.log 2>/dev/null | wc -l
echo "   Esperado: ≥2 archivos (baseline + under chaos)"

# 6. Alertas SLO
echo ""
echo "6. PrometheusRule aplicada:"
kubectl get prometheusrule microservices-slo-rules -n monitoring -o name 2>/dev/null && echo "   OK" || echo "   FALTA"

# 7. Runbook
echo ""
echo "7. Runbook creado:"
test -f docs/runbooks/inventory-service-unavailable.md && echo "   OK" || echo "   FALTA"

# 8. Postmortem
echo ""
echo "8. Postmortem creado:"
ls docs/postmortems/*.md 2>/dev/null | wc -l
echo "   Esperado: 1 archivo"

# 9. Estado del sistema
echo ""
echo "9. Estado actual del sistema:"
kubectl get pods -n microservices-prod --no-headers | awk '{print $1, $3}'

echo ""
echo "=== FIN DE VALIDACIÓN ==="
```

---

## Solución de Problemas

### Problema 1: k6 no puede conectar a los servicios (connection refused)

**Síntomas:** Al ejecutar k6, todos los checks fallan con `dial tcp 127.0.0.1:8001: connect: connection refused`.

**Causa:** Los port-forwards se cerraron o nunca se establecieron. Esto ocurre frecuentemente cuando los procesos de port-forward en background son terminados por el shell o cuando los pods se recrean durante experimentos de caos.

**Solución:**

```bash
# Matar port-forwards existentes
pkill -f "kubectl port-forward" 2>/dev/null

# Esperar a que los pods estén Ready
kubectl wait --for=condition=Ready pods -l app=order-service -n microservices-prod --timeout=60s
kubectl wait --for=condition=Ready pods -l app=inventory-service -n microservices-prod --timeout=60s

# Re-establecer port-forwards
kubectl port-forward svc/order-service -n microservices-prod 8001:8001 &
kubectl port-forward svc/inventory-service -n microservices-prod 8002:8002 &

# Verificar
sleep 3
curl -s http://localhost:8001/health && echo " OK" || echo " FAIL"
curl -s http://localhost:8002/health && echo " OK" || echo " FAIL"
```

---

### Problema 2: Chaos Mesh no puede inyectar fallos (permission denied / no pods selected)

**Síntomas:** El CRD de Chaos Mesh se aplica sin error pero `kubectl describe podchaos <name> -n chaos-testing` muestra `no pods selected` o eventos de permisos denegados.

**Causa:** Los label selectors en el manifiesto de caos no coinciden con los labels reales de los pods, o Chaos Mesh no tiene permisos RBAC para operar en el namespace `microservices-prod`.

**Solución:**

```bash
# 1. Verificar labels reales de los pods
kubectl get pods -n microservices-prod --show-labels

# 2. Ajustar el selector en el YAML si los labels difieren
# Por ejemplo, si el label es "app.kubernetes.io/name" en vez de "app":
# labelSelectors:
#   app.kubernetes.io/name: order-service

# 3. Verificar RBAC de Chaos Mesh
kubectl get clusterrole chaos-mesh-chaos-controller-manager-target-namespace -o yaml | grep -A5 "microservices-prod"

# 4. Si falta el permiso, agregar el namespace:
kubectl label ns microservices-prod chaos-mesh.org/inject=enabled --overwrite

# 5. Reiniciar el controller de Chaos Mesh
kubectl rollout restart deployment chaos-controller-manager -n chaos-testing
kubectl rollout status deployment chaos-controller-manager -n chaos-testing --timeout=60s

# 6. Re-aplicar el experimento
kubectl delete -f tests/chaos/pod-kill-order.yaml 2>/dev/null
kubectl apply -f tests/chaos/pod-kill-order.yaml

# 7. Verificar que selecciona pods
kubectl describe podchaos pod-kill-order-service -n chaos-testing | grep -A3 "Status"
```

---

## Limpieza

```bash
# Eliminar todos los experimentos de Chaos Mesh activos
kubectl delete podchaos --all -n chaos-testing
kubectl delete networkchaos --all -n chaos-testing
kubectl delete stresschaos --all -n chaos-testing

# Detener port-forwards
pkill -f "kubectl port-forward" 2>/dev/null

# Verificar que los servicios están saludables post-limpieza
kubectl get pods -n microservices-prod
kubectl rollout status deployment/order-service -n microservices-prod --timeout=60s
kubectl rollout status deployment/inventory-service -n microservices-prod --timeout=60s

# Nota: NO eliminar los archivos de tests/, docs/ ni monitoring/
# Estos son entregables del laboratorio
echo "Limpieza completada. Los artefactos del laboratorio se conservan en:"
echo "  - tests/load/        (scripts k6 y resultados)"
echo "  - tests/chaos/       (manifiestos y logs)"
echo "  - tests/contracts/   (resultados schemathesis)"
echo "  - docs/runbooks/     (runbook operacional)"
echo "  - docs/postmortems/  (postmortem estructurado)"
echo "  - monitoring/alerts/ (reglas SLO)"
```

---

## Resumen

En este laboratorio has completado un ciclo completo de validación de resiliencia:

| Fase | Logro |
|------|-------|
| **Pruebas de carga** | 4 escenarios k6 (smoke, load, stress, spike) con métricas exportadas a Prometheus |
| **Chaos Engineering** | 4 experimentos (pod-kill, network delay, partition, CPU stress) ejecutados con medición de impacto |
| **Contract Testing** | Validación de integridad de APIs durante condiciones de caos con schemathesis |
| **SLOs y Alertas** | Definición formal de SLOs con recording rules y alertas en Alertmanager |
| **Documentación operacional** | Runbook ejecutable y postmortem estructurado siguiendo plantilla Google SRE |

### Conceptos Clave Reforzados

- Los **SLOs** deben ser medibles, con error budgets que cuantifiquen la tolerancia a fallos
- El **chaos engineering** no busca romper el sistema sino descubrir debilidades antes de que impacten usuarios reales
- Los **postmortems** son blameless y se centran en mejoras sistémicas, no en culpas individuales
- El **contract testing** complementa las pruebas de carga validando la integridad semántica de las APIs

### Recursos Adicionales

- [k6 Documentation — Scenarios](https://grafana.com/docs/k6/latest/using-k6/scenarios/)
- [Chaos Mesh User Guide](https://chaos-mesh.org/docs/)
- [Google SRE Book — Postmortem Culture](https://sre.google/sre-book/postmortem-culture/)
- [Prometheus Alerting Rules](https://prometheus.io/docs/prometheus/latest/configuration/alerting_rules/)
- [Schemathesis Documentation](https://schemathesis.readthedocs.io/)
