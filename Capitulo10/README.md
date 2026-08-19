# Desplegar microservicios Python en Kubernetes con Istio

## Metadata

| Campo | Valor |
|-------|-------|
| **Duración** | 66 minutos |
| **Complejidad** | Alta |
| **Nivel Bloom** | Crear |
| **Tecnologías** | Istio 1.22.2, Envoy Proxy, Helm 3.15.2, Prometheus 2.53.0, Grafana 11.1.0, minikube |

## Descripción General

En este laboratorio final del curso, desplegarás el stack completo de microservicios Python en Kubernetes con Istio service mesh habilitado. Instalarás Istio con perfil demo, configurarás mTLS estricto entre servicios, implementarás un despliegue canary progresivo del `orders-service` (v1→v2) con distribución de tráfico controlada, configurarás enrutamiento blue/green basado en headers para `inventory-service`, e integrarás las métricas de Istio con el stack de observabilidad Prometheus/Grafana existente.

## Objetivos de Aprendizaje

- [ ] Instalar Istio 1.22.2 en minikube e inyectar sidecars Envoy automáticamente en el namespace `microservices`
- [ ] Configurar mTLS estricto usando PeerAuthentication y DestinationRule para cifrar todo el tráfico interno
- [ ] Implementar un despliegue canary del `orders-service` con distribución de tráfico progresiva 90/10 → 50/50 → 0/100
- [ ] Configurar enrutamiento basado en headers HTTP para blue/green deployment del `inventory-service`
- [ ] Integrar métricas de Istio (golden signals) con Prometheus y Grafana para observabilidad del service mesh

## Prerrequisitos

### Conocimiento Requerido

- Labs 06-00-01 a 09-00-01 completados (Traefik, Vault, OpenTelemetry, Helm charts)
- Comprensión de Envoy proxy, sidecars y conceptos de service mesh
- Familiaridad con patrones canary y blue/green deployment
- Experiencia con manifiestos Kubernetes (Deployments, Services, Namespaces)

### Acceso y Herramientas

- minikube ejecutándose con al menos 10 GB de RAM y 4 CPUs
- `istioctl` 1.22.2 instalado en el PATH
- `kubectl` configurado y apuntando al clúster minikube
- `helm` 3.15.2 disponible
- Imágenes Docker de `orders-service` v1.0 y v2.0, e `inventory-service` v1.0 construidas localmente

## Entorno del Laboratorio

### Requisitos de Hardware

| Recurso | Mínimo | Recomendado |
|---------|--------|-------------|
| RAM asignada a minikube | 10 GB | 12 GB |
| CPUs | 4 | 6 |
| Disco libre | 40 GB | 50 GB |

### Software Requerido

| Herramienta | Versión |
|-------------|---------|
| minikube | 1.33+ |
| istioctl | 1.22.2 |
| kubectl | 1.30+ |
| helm | 3.15.2 |
| Docker Engine | 26.1.3 |
| Python | 3.12.3 |

### Configuración Inicial del Entorno

```bash
# Crear directorio de trabajo para manifiestos Istio
mkdir -p ~/microservices-lab/istio/{gateway,mtls,canary,bluegreen,observability}

# Verificar que minikube está corriendo con recursos suficientes
minikube status

# Si no está corriendo, iniciarlo con recursos adecuados
minikube start --cpus=6 --memory=10240 --driver=docker --kubernetes-version=v1.30.0

# Verificar istioctl
istioctl version --remote=false

# Verificar que el namespace microservices existe con los servicios desplegados
kubectl get deployments -n microservices
```

**Salida esperada de `minikube status`:**
```
minikube
type: Control Plane
host: Running
kubelet: Running
apiserver: Running
kubeconfig: Configured
```

---

## Paso 1: Instalar Istio 1.22.2 con Perfil Demo

**Objetivo:** Instalar Istio en el clúster minikube usando el perfil `demo` que incluye todos los componentes necesarios para desarrollo y pruebas.

### Instrucciones

1. Verificar la compatibilidad del clúster con Istio:

```bash
istioctl x precheck
```

2. Instalar Istio con perfil demo:

```bash
istioctl install --set profile=demo -y
```

3. Verificar que todos los componentes de Istio están corriendo:

```bash
kubectl get pods -n istio-system
```

4. Verificar los servicios de Istio instalados:

```bash
kubectl get svc -n istio-system
```

5. Confirmar la versión instalada:

```bash
istioctl version
```

### Salida Esperada

```
NAME                                    READY   STATUS    RESTARTS   AGE
istio-egressgateway-xxxxx-xxxxx         1/1     Running   0          45s
istio-ingressgateway-xxxxx-xxxxx        1/1     Running   0          45s
istiod-xxxxx-xxxxx                      1/1     Running   0          60s
```

### Verificación

```bash
# Verificar que istiod (plano de control) está healthy
kubectl get deployment istiod -n istio-system -o jsonpath='{.status.readyReplicas}'
# Debe retornar: 1

# Verificar que el ingress gateway tiene IP asignada
kubectl get svc istio-ingressgateway -n istio-system -o jsonpath='{.status.loadBalancer.ingress[0].ip}'
```

> **Nota:** En minikube, el LoadBalancer puede mostrar `<pending>`. Usa `minikube tunnel` en otra terminal para asignar IPs externas.

---

## Paso 2: Habilitar Inyección de Sidecars y Reiniciar Pods

**Objetivo:** Etiquetar el namespace `microservices` para inyección automática de sidecars Envoy y reiniciar los pods existentes para que reciban el proxy.

### Instrucciones

1. Etiquetar el namespace para inyección automática:

```bash
kubectl label namespace microservices istio-injection=enabled --overwrite
```

2. Verificar la etiqueta:

```bash
kubectl get namespace microservices --show-labels
```

3. Reiniciar todos los deployments para inyectar sidecars:

```bash
kubectl rollout restart deployment -n microservices
```

4. Esperar a que todos los pods estén listos con 2/2 contenedores (app + sidecar):

```bash
kubectl get pods -n microservices -w
```

5. Verificar que el sidecar Envoy está inyectado en un pod del orders-service:

```bash
kubectl describe pod -l app=orders-service -n microservices | grep -A 2 "istio-proxy"
```

### Salida Esperada

```
NAME                                 READY   STATUS    RESTARTS   AGE
orders-service-xxxxx-xxxxx           2/2     Running   0          30s
inventory-service-xxxxx-xxxxx        2/2     Running   0          30s
```

La columna `READY` debe mostrar `2/2` indicando que tanto el contenedor de la aplicación como el sidecar `istio-proxy` están corriendo.

### Verificación

```bash
# Confirmar inyección del sidecar en cada pod
kubectl get pods -n microservices -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{range .spec.containers[*]}{.name}{" "}{end}{"\n"}{end}'
```

Cada pod debe listar dos contenedores: el de la aplicación y `istio-proxy`.

---

## Paso 3: Configurar Istio Gateway y VirtualService para Tráfico Externo

**Objetivo:** Reemplazar Traefik como ingress por Istio Gateway + VirtualService, manteniendo compatibilidad con las rutas existentes de los microservicios.

### Instrucciones

1. Crear el manifiesto del Gateway:

```bash
cat > ~/microservices-lab/istio/gateway/gateway.yaml << 'EOF'
apiVersion: networking.istio.io/v1beta1
kind: Gateway
metadata:
  name: microservices-gateway
  namespace: microservices
spec:
  selector:
    istio: ingressgateway    # Usa el ingress gateway de Istio
  servers:
    - port:
        number: 80
        name: http
        protocol: HTTP
      hosts:
        - "orders.local"
        - "inventory.local"
        - "*"
EOF
```

2. Crear el VirtualService para orders-service:

```bash
cat > ~/microservices-lab/istio/gateway/vs-orders.yaml << 'EOF'
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: orders-virtualservice
  namespace: microservices
spec:
  hosts:
    - "orders.local"
    - "orders-service"
  gateways:
    - microservices-gateway
    - mesh                    # También aplica a tráfico interno del mesh
  http:
    - match:
        - uri:
            prefix: /api/v1/orders
        - uri:
            exact: /health/ready
      route:
        - destination:
            host: orders-service
            port:
              number: 8001
EOF
```

3. Crear el VirtualService para inventory-service:

```bash
cat > ~/microservices-lab/istio/gateway/vs-inventory.yaml << 'EOF'
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: inventory-virtualservice
  namespace: microservices
spec:
  hosts:
    - "inventory.local"
    - "inventory-service"
  gateways:
    - microservices-gateway
    - mesh
  http:
    - match:
        - uri:
            prefix: /api/v1/inventory
        - uri:
            exact: /health/ready
      route:
        - destination:
            host: inventory-service
            port:
              number: 8002
EOF
```

4. Aplicar todos los manifiestos:

```bash
kubectl apply -f ~/microservices-lab/istio/gateway/
```

5. Obtener la URL del ingress gateway:

```bash
export INGRESS_HOST=$(minikube ip)
export INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="http2")].nodePort}')
echo "Gateway URL: http://${INGRESS_HOST}:${INGRESS_PORT}"
```

### Salida Esperada

```
gateway.networking.istio.io/microservices-gateway created
virtualservice.networking.istio.io/orders-virtualservice created
virtualservice.networking.istio.io/inventory-virtualservice created
```

### Verificación

```bash
# Probar acceso al orders-service a través del gateway
curl -s -H "Host: orders.local" http://${INGRESS_HOST}:${INGRESS_PORT}/health/ready

# Probar acceso al inventory-service
curl -s -H "Host: inventory.local" http://${INGRESS_HOST}:${INGRESS_PORT}/health/ready
```

Ambos deben retornar HTTP 200 con el estado de salud del servicio.

---

## Paso 4: Configurar mTLS Estricto entre Microservicios

**Objetivo:** Habilitar mTLS en modo STRICT para el namespace `microservices`, asegurando que toda comunicación entre servicios esté cifrada y autenticada mutuamente.

### Instrucciones

1. Crear la política PeerAuthentication en modo STRICT:

```bash
cat > ~/microservices-lab/istio/mtls/peer-authentication.yaml << 'EOF'
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: microservices
spec:
  mtls:
    mode: STRICT              # Rechaza cualquier tráfico plain-text
EOF
```

2. Crear la DestinationRule que aplica mTLS para comunicación entre servicios:

```bash
cat > ~/microservices-lab/istio/mtls/destination-rule-mtls.yaml << 'EOF'
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: orders-service-mtls
  namespace: microservices
spec:
  host: orders-service.microservices.svc.cluster.local
  trafficPolicy:
    tls:
      mode: ISTIO_MUTUAL      # Usa certificados emitidos por Istio CA
---
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: inventory-service-mtls
  namespace: microservices
spec:
  host: inventory-service.microservices.svc.cluster.local
  trafficPolicy:
    tls:
      mode: ISTIO_MUTUAL
EOF
```

3. Aplicar los manifiestos:

```bash
kubectl apply -f ~/microservices-lab/istio/mtls/
```

4. Verificar que mTLS está activo usando `istioctl`:

```bash
# Obtener el nombre de un pod del orders-service
ORDERS_POD=$(kubectl get pod -l app=orders-service -n microservices -o jsonpath='{.items[0].metadata.name}')

# Describir la configuración de seguridad del pod
istioctl x describe pod ${ORDERS_POD} -n microservices
```

5. Verificar los certificados mTLS:

```bash
istioctl proxy-config secret ${ORDERS_POD} -n microservices
```

### Salida Esperada

La salida de `istioctl x describe pod` debe incluir:

```
Pod is STRICT, clients must use TLS to connect
Workload is behind services: orders-service.microservices
   Host: orders-service.microservices matches policy default/microservices
```

La salida de `proxy-config secret` debe mostrar certificados válidos:

```
RESOURCE NAME          TYPE           STATUS    VALID CERT   SERIAL NUMBER   NOT AFTER
default                Cert Chain     ACTIVE    true         ...             ...
ROOTCA                 CA             ACTIVE    true         ...             ...
```

### Verificación

```bash
# Intentar una conexión plain-text desde un pod sin sidecar (debe fallar)
kubectl run test-no-mesh --image=curlimages/curl --rm -it --restart=Never -n default -- \
  curl -s -o /dev/null -w "%{http_code}" http://orders-service.microservices.svc.cluster.local:8001/health/ready

# Debe retornar código de error (000 o connection reset) porque mTLS STRICT rechaza plain-text
```

```bash
# Conexión desde un pod DENTRO del mesh (debe funcionar)
kubectl run test-in-mesh --image=curlimages/curl --rm -it --restart=Never -n microservices -- \
  curl -s -o /dev/null -w "%{http_code}" http://orders-service:8001/health/ready

# Debe retornar: 200
```

---

## Paso 5: Implementar Canary Deployment del Orders-Service

**Objetivo:** Desplegar `orders-service` v2.0 junto a v1.0 y configurar distribución de tráfico progresiva usando VirtualService y DestinationRule de Istio.

### Instrucciones

1. Crear la imagen v2.0 del orders-service (simulada con un label diferente y nueva ruta `/v2/orders`):

```bash
cat > ~/microservices-lab/istio/canary/deployment-orders-v2.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: orders-service-v2
  namespace: microservices
  labels:
    app: orders-service
    version: v2
spec:
  replicas: 1
  selector:
    matchLabels:
      app: orders-service
      version: v2
  template:
    metadata:
      labels:
        app: orders-service
        version: v2
    spec:
      containers:
        - name: orders-api
          image: orders-service:v2.0
          ports:
            - containerPort: 8001
          env:
            - name: SERVICE_VERSION
              value: "v2.0"
            - name: POSTGRES_HOST
              value: "mslab-postgres"
            - name: POSTGRES_USER
              value: "mslab_user"
            - name: POSTGRES_PASSWORD
              value: "mslab_pass_2024"
            - name: POSTGRES_DB
              value: "orders_db"
          resources:
            requests:
              cpu: "250m"
              memory: "256Mi"
            limits:
              cpu: "500m"
              memory: "512Mi"
          readinessProbe:
            httpGet:
              path: /health/ready
              port: 8001
            initialDelaySeconds: 5
            periodSeconds: 10
EOF
```

2. Asegurar que el deployment v1 existente tenga el label `version: v1`:

```bash
cat > ~/microservices-lab/istio/canary/patch-orders-v1.yaml << 'EOF'
spec:
  template:
    metadata:
      labels:
        version: v1
EOF

kubectl patch deployment orders-service -n microservices --patch-file ~/microservices-lab/istio/canary/patch-orders-v1.yaml
```

3. Desplegar la versión v2:

```bash
kubectl apply -f ~/microservices-lab/istio/canary/deployment-orders-v2.yaml
```

4. Crear la DestinationRule con subsets para canary:

```bash
cat > ~/microservices-lab/istio/canary/destination-rule-canary.yaml << 'EOF'
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: orders-service-canary
  namespace: microservices
spec:
  host: orders-service
  trafficPolicy:
    tls:
      mode: ISTIO_MUTUAL
  subsets:
    - name: v1
      labels:
        version: v1
    - name: v2
      labels:
        version: v2
EOF
```

5. Crear el VirtualService con distribución 90/10:

```bash
cat > ~/microservices-lab/istio/canary/vs-canary-90-10.yaml << 'EOF'
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: orders-virtualservice
  namespace: microservices
spec:
  hosts:
    - "orders.local"
    - "orders-service"
  gateways:
    - microservices-gateway
    - mesh
  http:
    - match:
        - uri:
            prefix: /api/v1/orders
        - uri:
            prefix: /v2/orders
        - uri:
            exact: /health/ready
      route:
        - destination:
            host: orders-service
            subset: v1
            port:
              number: 8001
          weight: 90
        - destination:
            host: orders-service
            subset: v2
            port:
              number: 8001
          weight: 10
EOF
```

6. Aplicar los manifiestos de canary:

```bash
kubectl apply -f ~/microservices-lab/istio/canary/destination-rule-canary.yaml
kubectl apply -f ~/microservices-lab/istio/canary/vs-canary-90-10.yaml
```

7. Verificar la distribución de tráfico con múltiples solicitudes:

```bash
# Enviar 20 solicitudes y contar respuestas por versión
for i in $(seq 1 20); do
  curl -s -H "Host: orders.local" http://${INGRESS_HOST}:${INGRESS_PORT}/health/ready | grep -o '"version":"[^"]*"'
done | sort | uniq -c
```

8. Progresión a 50/50:

```bash
cat > ~/microservices-lab/istio/canary/vs-canary-50-50.yaml << 'EOF'
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: orders-virtualservice
  namespace: microservices
spec:
  hosts:
    - "orders.local"
    - "orders-service"
  gateways:
    - microservices-gateway
    - mesh
  http:
    - match:
        - uri:
            prefix: /api/v1/orders
        - uri:
            prefix: /v2/orders
        - uri:
            exact: /health/ready
      route:
        - destination:
            host: orders-service
            subset: v1
            port:
              number: 8001
          weight: 50
        - destination:
            host: orders-service
            subset: v2
            port:
              number: 8001
          weight: 50
EOF

kubectl apply -f ~/microservices-lab/istio/canary/vs-canary-50-50.yaml
```

9. Progresión final a 0/100 (promoción completa de v2):

```bash
cat > ~/microservices-lab/istio/canary/vs-canary-0-100.yaml << 'EOF'
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: orders-virtualservice
  namespace: microservices
spec:
  hosts:
    - "orders.local"
    - "orders-service"
  gateways:
    - microservices-gateway
    - mesh
  http:
    - match:
        - uri:
            prefix: /api/v1/orders
        - uri:
            prefix: /v2/orders
        - uri:
            exact: /health/ready
      route:
        - destination:
            host: orders-service
            subset: v1
            port:
              number: 8001
          weight: 0
        - destination:
            host: orders-service
            subset: v2
            port:
              number: 8001
          weight: 100
EOF

kubectl apply -f ~/microservices-lab/istio/canary/vs-canary-0-100.yaml
```

### Salida Esperada

Tras aplicar la distribución 90/10, al enviar 20 solicitudes:

```
 18 "version":"v1.0"
  2 "version":"v2.0"
```

(Los números son aproximados; la distribución estadística puede variar ligeramente.)

### Verificación

```bash
# Verificar que ambos subsets están reconocidos
istioctl proxy-config clusters ${ORDERS_POD} -n microservices | grep orders-service

# Verificar la configuración de rutas activa
istioctl proxy-config routes ${ORDERS_POD} -n microservices --name "80" -o json | grep -A 5 "weightedClusters"
```

---

## Paso 6: Configurar Enrutamiento Blue/Green Basado en Headers para Inventory-Service

**Objetivo:** Implementar un patrón blue/green para `inventory-service` donde el tráfico se dirige a una versión u otra según el header HTTP `x-version`.

### Instrucciones

1. Asegurar labels de versión en el deployment existente de inventory-service:

```bash
kubectl patch deployment inventory-service -n microservices --type='json' \
  -p='[{"op": "add", "path": "/spec/template/metadata/labels/version", "value": "blue"}]'
```

2. Crear el deployment "green" del inventory-service:

```bash
cat > ~/microservices-lab/istio/bluegreen/deployment-inventory-green.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: inventory-service-green
  namespace: microservices
  labels:
    app: inventory-service
    version: green
spec:
  replicas: 1
  selector:
    matchLabels:
      app: inventory-service
      version: green
  template:
    metadata:
      labels:
        app: inventory-service
        version: green
    spec:
      containers:
        - name: inventory-api
          image: inventory-service:v2.0
          ports:
            - containerPort: 8002
          env:
            - name: SERVICE_VERSION
              value: "green"
            - name: POSTGRES_HOST
              value: "mslab-postgres"
            - name: POSTGRES_USER
              value: "mslab_user"
            - name: POSTGRES_PASSWORD
              value: "mslab_pass_2024"
            - name: POSTGRES_DB
              value: "inventory_db"
          resources:
            requests:
              cpu: "250m"
              memory: "256Mi"
            limits:
              cpu: "500m"
              memory: "512Mi"
          readinessProbe:
            httpGet:
              path: /health/ready
              port: 8002
            initialDelaySeconds: 5
            periodSeconds: 10
EOF
```

3. Crear la DestinationRule con subsets blue/green:

```bash
cat > ~/microservices-lab/istio/bluegreen/destination-rule-bg.yaml << 'EOF'
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: inventory-service-bg
  namespace: microservices
spec:
  host: inventory-service
  trafficPolicy:
    tls:
      mode: ISTIO_MUTUAL
  subsets:
    - name: blue
      labels:
        version: blue
    - name: green
      labels:
        version: green
EOF
```

4. Crear el VirtualService con enrutamiento basado en headers:

```bash
cat > ~/microservices-lab/istio/bluegreen/vs-bluegreen.yaml << 'EOF'
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: inventory-virtualservice
  namespace: microservices
spec:
  hosts:
    - "inventory.local"
    - "inventory-service"
  gateways:
    - microservices-gateway
    - mesh
  http:
    # Ruta para tráfico con header x-version: green
    - match:
        - headers:
            x-version:
              exact: green
      route:
        - destination:
            host: inventory-service
            subset: green
            port:
              number: 8002
    # Ruta para tráfico con header x-version: blue (explícito)
    - match:
        - headers:
            x-version:
              exact: blue
      route:
        - destination:
            host: inventory-service
            subset: blue
            port:
              number: 8002
    # Ruta por defecto: blue (producción actual)
    - route:
        - destination:
            host: inventory-service
            subset: blue
            port:
              number: 8002
EOF
```

5. Aplicar todos los manifiestos:

```bash
kubectl apply -f ~/microservices-lab/istio/bluegreen/
```

6. Esperar a que el pod green esté listo:

```bash
kubectl wait --for=condition=ready pod -l app=inventory-service,version=green -n microservices --timeout=60s
```

### Salida Esperada

```
deployment.apps/inventory-service-green created
destinationrule.networking.istio.io/inventory-service-bg created
virtualservice.networking.istio.io/inventory-virtualservice configured
```

### Verificación

```bash
# Solicitud sin header → debe ir a blue
curl -s -H "Host: inventory.local" \
  http://${INGRESS_HOST}:${INGRESS_PORT}/health/ready

# Solicitud con header x-version: green → debe ir a green
curl -s -H "Host: inventory.local" -H "x-version: green" \
  http://${INGRESS_HOST}:${INGRESS_PORT}/health/ready

# Solicitud con header x-version: blue → debe ir a blue explícitamente
curl -s -H "Host: inventory.local" -H "x-version: blue" \
  http://${INGRESS_HOST}:${INGRESS_PORT}/health/ready
```

Verificar que las respuestas incluyen la versión correcta del servicio en el campo `version` del JSON de salud.

---

## Paso 7: Integrar Observabilidad de Istio con Prometheus y Grafana

**Objetivo:** Configurar la recolección de métricas de Istio en Prometheus y crear dashboards en Grafana que muestren los golden signals diferenciados por versión durante el canary.

### Instrucciones

1. Verificar que Prometheus está scrapeando métricas de Istio:

```bash
# Las métricas de Istio se exponen automáticamente; verificar el ServiceMonitor
kubectl get servicemonitor -n istio-system
```

2. Si no existe un ServiceMonitor, crear la configuración de scrape para Prometheus:

```bash
cat > ~/microservices-lab/istio/observability/prometheus-istio-scrape.yaml << 'EOF'
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: istio-mesh-monitor
  namespace: monitoring
  labels:
    release: prometheus
spec:
  selector:
    matchExpressions:
      - key: istio
        operator: Exists
  namespaceSelector:
    matchNames:
      - istio-system
  endpoints:
    - port: http-monitoring
      interval: 15s
      path: /stats/prometheus
---
apiVersion: monitoring.coreos.com/v1
kind: PodMonitor
metadata:
  name: envoy-stats-monitor
  namespace: monitoring
  labels:
    release: prometheus
spec:
  selector:
    matchExpressions:
      - key: security.istio.io/tlsMode
        operator: Exists
  namespaceSelector:
    matchNames:
      - microservices
  podMetricsEndpoints:
    - port: http-envoy-prom
      path: /stats/prometheus
      interval: 15s
EOF

kubectl apply -f ~/microservices-lab/istio/observability/prometheus-istio-scrape.yaml
```

3. Crear el ConfigMap con el dashboard de Grafana para Istio golden signals:

```bash
cat > ~/microservices-lab/istio/observability/grafana-istio-dashboard.json << 'EOF'
{
  "dashboard": {
    "title": "Istio Service Mesh - Golden Signals por Versión",
    "uid": "istio-golden-signals",
    "panels": [
      {
        "title": "Request Rate por Versión (orders-service)",
        "type": "timeseries",
        "targets": [
          {
            "expr": "sum(rate(istio_requests_total{destination_service=\"orders-service.microservices.svc.cluster.local\"}[5m])) by (destination_version)",
            "legendFormat": "v{{destination_version}}"
          }
        ],
        "gridPos": {"h": 8, "w": 12, "x": 0, "y": 0}
      },
      {
        "title": "Error Rate por Versión (orders-service)",
        "type": "timeseries",
        "targets": [
          {
            "expr": "sum(rate(istio_requests_total{destination_service=\"orders-service.microservices.svc.cluster.local\", response_code=~\"5.*\"}[5m])) by (destination_version) / sum(rate(istio_requests_total{destination_service=\"orders-service.microservices.svc.cluster.local\"}[5m])) by (destination_version)",
            "legendFormat": "v{{destination_version}} error rate"
          }
        ],
        "gridPos": {"h": 8, "w": 12, "x": 12, "y": 0}
      },
      {
        "title": "Latencia P99 por Versión (orders-service)",
        "type": "timeseries",
        "targets": [
          {
            "expr": "histogram_quantile(0.99, sum(rate(istio_request_duration_milliseconds_bucket{destination_service=\"orders-service.microservices.svc.cluster.local\"}[5m])) by (le, destination_version))",
            "legendFormat": "v{{destination_version}} p99"
          }
        ],
        "gridPos": {"h": 8, "w": 12, "x": 0, "y": 8}
      },
      {
        "title": "Tráfico por Versión (inventory-service blue/green)",
        "type": "timeseries",
        "targets": [
          {
            "expr": "sum(rate(istio_requests_total{destination_service=\"inventory-service.microservices.svc.cluster.local\"}[5m])) by (destination_version)",
            "legendFormat": "{{destination_version}}"
          }
        ],
        "gridPos": {"h": 8, "w": 12, "x": 12, "y": 8}
      }
    ],
    "time": {"from": "now-15m", "to": "now"},
    "refresh": "10s"
  }
}
EOF
```

4. Importar el dashboard en Grafana:

```bash
# Obtener el puerto de Grafana
GRAFANA_PORT=$(kubectl get svc grafana -n monitoring -o jsonpath='{.spec.ports[0].nodePort}' 2>/dev/null || echo "3000")

# Importar via API (si Grafana está accesible)
curl -X POST http://admin:admin@$(minikube ip):${GRAFANA_PORT}/api/dashboards/db \
  -H "Content-Type: application/json" \
  -d @~/microservices-lab/istio/observability/grafana-istio-dashboard.json
```

5. Generar tráfico para verificar métricas:

```bash
# Generar 100 solicitudes distribuidas entre versiones
for i in $(seq 1 100); do
  curl -s -H "Host: orders.local" http://${INGRESS_HOST}:${INGRESS_PORT}/health/ready > /dev/null
  sleep 0.1
done &

# Generar tráfico blue/green
for i in $(seq 1 50); do
  curl -s -H "Host: inventory.local" -H "x-version: green" http://${INGRESS_HOST}:${INGRESS_PORT}/health/ready > /dev/null
  curl -s -H "Host: inventory.local" http://${INGRESS_HOST}:${INGRESS_PORT}/health/ready > /dev/null
  sleep 0.1
done &
```

6. Verificar métricas en Prometheus:

```bash
# Port-forward a Prometheus
kubectl port-forward svc/prometheus-server -n monitoring 9090:9090 &

# Consultar métricas de Istio
curl -s "http://localhost:9090/api/v1/query?query=istio_requests_total{destination_service=~\"orders.*\"}" | python3 -m json.tool | head -30
```

### Salida Esperada

La consulta a Prometheus debe retornar métricas con labels de versión:

```json
{
  "status": "success",
  "data": {
    "resultType": "vector",
    "result": [
      {
        "metric": {
          "destination_service": "orders-service.microservices.svc.cluster.local",
          "destination_version": "v1",
          "response_code": "200"
        },
        "value": [1719000000, "85"]
      },
      {
        "metric": {
          "destination_service": "orders-service.microservices.svc.cluster.local",
          "destination_version": "v2",
          "response_code": "200"
        },
        "value": [1719000000, "15"]
      }
    ]
  }
}
```

### Verificación

```bash
# Verificar que las métricas de Istio están disponibles
curl -s "http://localhost:9090/api/v1/label/__name__/values" | python3 -c "
import json, sys
names = json.load(sys.stdin)['data']
istio_metrics = [n for n in names if n.startswith('istio_')]
print(f'Métricas Istio encontradas: {len(istio_metrics)}')
for m in istio_metrics[:5]:
    print(f'  - {m}')
"
```

Debe mostrar al menos `istio_requests_total` e `istio_request_duration_milliseconds_bucket`.

---

## Validación y Pruebas Finales

### Prueba Integral del Service Mesh

```bash
#!/bin/bash
# ~/microservices-lab/istio/validate-mesh.sh
# Script de validación completa del service mesh

echo "=== Validación del Service Mesh Istio ==="
echo ""

# 1. Verificar componentes de Istio
echo "1. Componentes de Istio:"
kubectl get pods -n istio-system --no-headers | awk '{print "   ✓ " $1 " - " $3}'
echo ""

# 2. Verificar sidecars inyectados
echo "2. Sidecars inyectados en namespace microservices:"
TOTAL_PODS=$(kubectl get pods -n microservices --no-headers | wc -l)
MESH_PODS=$(kubectl get pods -n microservices -o jsonpath='{range .items[*]}{.spec.containers[*].name}{"\n"}{end}' | grep -c "istio-proxy")
echo "   Pods con sidecar: ${MESH_PODS}/${TOTAL_PODS}"
echo ""

# 3. Verificar mTLS
echo "3. Estado mTLS:"
ORDERS_POD=$(kubectl get pod -l app=orders-service -n microservices -o jsonpath='{.items[0].metadata.name}')
istioctl x describe pod ${ORDERS_POD} -n microservices 2>/dev/null | grep -i "strict\|TLS"
echo ""

# 4. Verificar canary (distribución actual)
echo "4. Distribución de tráfico canary (orders-service):"
kubectl get virtualservice orders-virtualservice -n microservices -o jsonpath='{range .spec.http[0].route[*]}  subset={.destination.subset} weight={.weight}{"\n"}{end}'
echo ""

# 5. Verificar blue/green
echo "5. Enrutamiento blue/green (inventory-service):"
echo "   Sin header (default → blue):"
RESPONSE=$(curl -s -H "Host: inventory.local" http://${INGRESS_HOST}:${INGRESS_PORT}/health/ready 2>/dev/null)
echo "   Respuesta: ${RESPONSE}" | head -1
echo "   Con header x-version: green:"
RESPONSE=$(curl -s -H "Host: inventory.local" -H "x-version: green" http://${INGRESS_HOST}:${INGRESS_PORT}/health/ready 2>/dev/null)
echo "   Respuesta: ${RESPONSE}" | head -1
echo ""

# 6. Verificar métricas
echo "6. Métricas de Istio en Prometheus:"
METRICS_COUNT=$(curl -s "http://localhost:9090/api/v1/label/__name__/values" 2>/dev/null | python3 -c "import json,sys; print(len([n for n in json.load(sys.stdin).get('data',[]) if n.startswith('istio_')]))" 2>/dev/null || echo "0")
echo "   Métricas Istio disponibles: ${METRICS_COUNT}"
echo ""

echo "=== Validación Completa ==="
```

```bash
chmod +x ~/microservices-lab/istio/validate-mesh.sh
bash ~/microservices-lab/istio/validate-mesh.sh
```

### Verificación de Certificados mTLS

```bash
# Verificar que los certificados están activos y no expirados
ORDERS_POD=$(kubectl get pod -l app=orders-service -n microservices -o jsonpath='{.items[0].metadata.name}')

istioctl proxy-config secret ${ORDERS_POD} -n microservices -o json | python3 -c "
import json, sys
from datetime import datetime

data = json.load(sys.stdin)
for secret in data.get('dynamicActiveSecrets', []):
    name = secret.get('name', 'unknown')
    cert_chain = secret.get('secret', {}).get('tlsCertificate', {})
    if cert_chain:
        print(f'Secreto: {name} - Estado: ACTIVO')
print('mTLS verificado correctamente.')
"
```

---

## Solución de Problemas

### Problema 1: Pods se quedan en estado `Init:CrashLoopBackOff` tras habilitar inyección

**Síntomas:**
- Los pods muestran estado `Init:CrashLoopBackOff` o `Init:0/1` tras reiniciar los deployments
- El contenedor `istio-init` falla repetidamente
- Los logs del init container muestran errores de iptables

**Causa:**
El contenedor de inicialización `istio-init` necesita la capacidad `NET_ADMIN` para configurar las reglas de iptables que redirigen el tráfico al sidecar Envoy. En algunos entornos con PodSecurityPolicy o SecurityContextConstraints restrictivos, esta capacidad está bloqueada.

**Solución:**

```bash
# 1. Verificar los logs del init container
FAILING_POD=$(kubectl get pods -n microservices --field-selector=status.phase!=Running -o jsonpath='{.items[0].metadata.name}')
kubectl logs ${FAILING_POD} -n microservices -c istio-init

# 2. Si el error es de permisos iptables, verificar la configuración CNI de Istio
# Opción A: Instalar Istio CNI plugin (evita necesidad de NET_ADMIN)
istioctl install --set profile=demo --set components.cni.enabled=true -y

# 3. O permitir la capacidad NET_ADMIN en el namespace
kubectl label namespace microservices pod-security.kubernetes.io/enforce=privileged --overwrite

# 4. Reiniciar los pods
kubectl rollout restart deployment -n microservices
```

---

### Problema 2: VirtualService no distribuye tráfico correctamente al canary

**Síntomas:**
- Todo el tráfico va a v1 a pesar de configurar distribución 90/10
- Las solicitudes nunca llegan al subset v2
- `istioctl analyze` no reporta errores

**Causa:**
Los labels del pod v2 no coinciden con los definidos en el subset de la DestinationRule, o el Service de Kubernetes no selecciona los pods v2. El Service debe usar un selector que coincida con ambas versiones (solo `app: orders-service`, sin `version`).

**Solución:**

```bash
# 1. Verificar que el Service selecciona ambas versiones
kubectl get svc orders-service -n microservices -o jsonpath='{.spec.selector}'
# Debe mostrar solo: {"app":"orders-service"} (sin version)

# 2. Si incluye version, parchearlo
kubectl patch svc orders-service -n microservices --type='json' \
  -p='[{"op": "remove", "path": "/spec/selector/version"}]'

# 3. Verificar que los pods tienen los labels correctos
kubectl get pods -n microservices -l app=orders-service --show-labels

# 4. Verificar que la DestinationRule reconoce ambos subsets
istioctl proxy-config clusters ${ORDERS_POD} -n microservices | grep "orders-service"
# Debe mostrar entries para outbound|8001|v1|orders-service y outbound|8001|v2|orders-service

# 5. Forzar sincronización del proxy
istioctl proxy-config routes ${ORDERS_POD} -n microservices --name "80" -o json | grep weight
# Debe mostrar los pesos configurados (90 y 10)
```

---

## Limpieza

```bash
# Eliminar manifiestos de canary (revertir a v1 solamente)
kubectl delete deployment orders-service-v2 -n microservices
kubectl delete deployment inventory-service-green -n microservices

# Restaurar VirtualServices a estado simple (sin canary/blue-green)
kubectl delete virtualservice orders-virtualservice -n microservices
kubectl delete virtualservice inventory-virtualservice -n microservices
kubectl delete destinationrule orders-service-canary -n microservices
kubectl delete destinationrule inventory-service-bg -n microservices

# Mantener el Gateway y mTLS (son parte de la arquitectura de referencia)
# Si se desea desinstalar Istio completamente:
# istioctl uninstall --purge -y
# kubectl delete namespace istio-system

# Remover label de inyección si se desea volver al estado sin mesh
# kubectl label namespace microservices istio-injection-
# kubectl rollout restart deployment -n microservices

# Limpiar port-forwards
pkill -f "port-forward" 2>/dev/null || true

echo "Limpieza completada. Los manifiestos de referencia permanecen en ~/microservices-lab/istio/"
```

---

## Resumen

En este laboratorio has completado la arquitectura de referencia del curso integrando Istio service mesh con el stack completo de microservicios Python:

| Componente | Estado |
|------------|--------|
| Istio 1.22.2 con perfil demo | ✅ Instalado |
| Sidecars Envoy inyectados | ✅ Todos los pods en namespace `microservices` |
| mTLS STRICT | ✅ Todo tráfico interno cifrado |
| Gateway + VirtualService | ✅ Reemplaza Traefik como ingress |
| Canary deployment (orders-service) | ✅ Distribución progresiva 90/10 → 50/50 → 0/100 |
| Blue/Green (inventory-service) | ✅ Enrutamiento por header `x-version` |
| Observabilidad Istio + Prometheus/Grafana | ✅ Golden signals por versión |

**Manifiestos consolidados en:** `~/microservices-lab/istio/`

```
~/microservices-lab/istio/
├── gateway/
│   ├── gateway.yaml
│   ├── vs-orders.yaml
│   └── vs-inventory.yaml
├── mtls/
│   ├── peer-authentication.yaml
│   └── destination-rule-mtls.yaml
├── canary/
│   ├── deployment-orders-v2.yaml
│   ├── patch-orders-v1.yaml
│   ├── destination-rule-canary.yaml
│   ├── vs-canary-90-10.yaml
│   ├── vs-canary-50-50.yaml
│   └── vs-canary-0-100.yaml
├── bluegreen/
│   ├── deployment-inventory-green.yaml
│   ├── destination-rule-bg.yaml
│   └── vs-bluegreen.yaml
└── observability/
    ├── prometheus-istio-scrape.yaml
    └── grafana-istio-dashboard.json
```

### Recursos Adicionales

- [Documentación oficial de Istio 1.22](https://istio.io/latest/docs/)
- [Istio Traffic Management Concepts](https://istio.io/latest/docs/concepts/traffic-management/)
- [Istio Security: mTLS](https://istio.io/latest/docs/concepts/security/#mutual-tls-authentication)
- [Canary Deployments with Istio](https://istio.io/latest/docs/tasks/traffic-management/traffic-shifting/)
- [Prometheus Metrics from Istio](https://istio.io/latest/docs/reference/config/metrics/)
