# 4 Laboratorio: desplegar Gateway y políticas para servicios Python

## Metadata

| Campo | Valor |
|-------|-------|
| **Duración** | 66 minutos |
| **Complejidad** | Media |
| **Nivel Bloom** | Aplicar |
| **Tecnologías** | Traefik 3.0.4, FastAPI 0.111.0, Kubernetes IngressRoute CRDs, Helm 3.15.2, Docker 26.1.4, minikube 1.33.1 |

## Visión General

En este laboratorio desplegarás Traefik 3.0.4 como API Gateway e Ingress Controller en un clúster minikube, configurarás rutas declarativas mediante IngressRoute CRDs hacia tres microservicios Python FastAPI, e implementarás políticas de rate limiting y circuit breaker en el edge. Este lab establece la infraestructura base del directorio `/home/student/msvc-labs/` que se reutilizará en laboratorios posteriores.

## Objetivos de Aprendizaje

- [ ] Desplegar Traefik 3.0.4 como Ingress Controller en minikube usando Helm con CRDs habilitados
- [ ] Configurar IngressRoute CRDs para enrutar tráfico hacia tres microservicios Python FastAPI bajo el dominio `microservices.local`
- [ ] Implementar rate limiting diferenciado por servicio usando middlewares declarativos de Traefik
- [ ] Configurar circuit breaker en el edge para proteger servicios backend ante fallos cascada
- [ ] Validar las políticas mediante pruebas de carga con curl y scripts Python

## Prerrequisitos

### Conocimiento Previo

- Conceptos de Kubernetes: Deployments, Services, Namespaces
- Familiaridad con Helm charts y valores de configuración
- Comprensión del rol del API Gateway y patrones de enrutamiento (proxy inverso)
- Experiencia básica con FastAPI y Docker

### Acceso y Herramientas

| Herramienta | Versión | Verificación |
|-------------|---------|--------------|
| minikube | 1.33.1 | `minikube version` |
| kubectl | 1.30.2 | `kubectl version --client` |
| Helm | 3.15.2 | `helm version --short` |
| Docker Engine | 26.1.4 | `docker --version` |
| Python | 3.12.3 | `python3 --version` |

## Entorno del Laboratorio

### Requisitos de Hardware

| Recurso | Mínimo | Recomendado |
|---------|--------|-------------|
| CPU | 4 núcleos | 8 núcleos |
| RAM | 16 GB | 32 GB |
| Disco SSD | 30 GB libres | 50 GB libres |
| Red | 10 Mbps | 50 Mbps |

### Estructura de Directorios

```
/home/student/msvc-labs/
├── services/
│   ├── orders-service/
│   ├── inventory-service/
│   └── users-service/
├── k8s/
│   ├── deployments/
│   ├── services/
│   └── traefik/
│       ├── middlewares/
│       └── ingress-routes/
├── scripts/
└── docker/
```

### Configuración Inicial del Entorno

```bash
# Crear estructura de directorios
mkdir -p /home/student/msvc-labs/{services/{orders-service,inventory-service,users-service},k8s/{deployments,services,traefik/{middlewares,ingress-routes}},scripts,docker}
cd /home/student/msvc-labs
```

---

## Paso 1: Iniciar clúster minikube y verificar estado

### Objetivo

Arrancar un clúster minikube con recursos suficientes para ejecutar Traefik y tres microservicios, deshabilitando el addon de ingress por defecto para evitar conflictos.

### Instrucciones

1. Detener cualquier clúster minikube existente y crear uno nuevo con recursos adecuados:

```bash
minikube delete --all 2>/dev/null
minikube start \
  --driver=docker \
  --cpus=4 \
  --memory=8192 \
  --disk-size=30g \
  --kubernetes-version=v1.30.2 \
  --addons=metrics-server \
  --profile=msvc-gateway
```

2. Verificar que el clúster está operativo:

```bash
kubectl cluster-info
kubectl get nodes -o wide
```

3. Deshabilitar el addon de ingress por defecto (si estuviera habilitado):

```bash
minikube addons disable ingress --profile=msvc-gateway 2>/dev/null || true
```

4. Crear los namespaces necesarios:

```bash
kubectl create namespace microservices
kubectl create namespace traefik
```

### Salida Esperada

```
😄  [msvc-gateway] minikube v1.33.1 on Ubuntu 22.04
✨  Using the docker driver based on user configuration
...
🏄  Done! kubectl is now configured to use "msvc-gateway" cluster
```

```
Kubernetes control plane is running at https://192.168.49.2:8443
namespace/microservices created
namespace/traefik created
```

### Verificación

```bash
kubectl get namespaces | grep -E "microservices|traefik"
```

Debe mostrar ambos namespaces en estado `Active`.

---

## Paso 2: Construir e importar las imágenes de los microservicios

### Objetivo

Crear tres microservicios Python FastAPI simples (orders, inventory, users), construir sus imágenes Docker e importarlas al registro interno de minikube.

### Instrucciones

1. Crear el archivo `requirements.txt` compartido:

```bash
cat > /home/student/msvc-labs/docker/requirements.txt << 'EOF'
fastapi==0.111.0
uvicorn==0.29.0
pydantic==2.7.1
EOF
```

2. Crear el código del **orders-service**:

```bash
cat > /home/student/msvc-labs/services/orders-service/main.py << 'EOF'
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from datetime import datetime
import os

app = FastAPI(title="Orders Service", version="1.0.0")

class Order(BaseModel):
    id: int
    product: str
    quantity: int
    status: str
    created_at: str

ORDERS_DB = {
    1: Order(id=1, product="Laptop Pro", quantity=1, status="confirmed", created_at="2024-06-01T10:00:00Z"),
    2: Order(id=2, product="Mouse Wireless", quantity=3, status="pending", created_at="2024-06-01T11:30:00Z"),
    3: Order(id=3, product="Monitor 27\"", quantity=1, status="shipped", created_at="2024-06-02T09:15:00Z"),
}

@app.get("/health")
async def health():
    return {"status": "healthy", "service": "orders-service", "version": "1.0.0"}

@app.get("/api/v1/orders")
async def list_orders():
    return {"orders": list(ORDERS_DB.values()), "total": len(ORDERS_DB)}

@app.get("/api/v1/orders/{order_id}")
async def get_order(order_id: int):
    if order_id not in ORDERS_DB:
        raise HTTPException(status_code=404, detail=f"Order {order_id} not found")
    return ORDERS_DB[order_id]

@app.get("/api/v1/orders/fail/simulate")
async def simulate_failure():
    """Endpoint para simular fallos y probar circuit breaker."""
    raise HTTPException(status_code=500, detail="Simulated internal error")
EOF
```

3. Crear el código del **inventory-service**:

```bash
cat > /home/student/msvc-labs/services/inventory-service/main.py << 'EOF'
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel

app = FastAPI(title="Inventory Service", version="1.0.0")

class InventoryItem(BaseModel):
    sku: str
    name: str
    quantity: int
    warehouse: str

INVENTORY_DB = {
    "SKU-001": InventoryItem(sku="SKU-001", name="Laptop Pro", quantity=42, warehouse="Madrid"),
    "SKU-002": InventoryItem(sku="SKU-002", name="Mouse Wireless", quantity=150, warehouse="Barcelona"),
    "SKU-003": InventoryItem(sku="SKU-003", name="Monitor 27\"", quantity=28, warehouse="Madrid"),
}

@app.get("/health")
async def health():
    return {"status": "healthy", "service": "inventory-service", "version": "1.0.0"}

@app.get("/api/v1/inventory")
async def list_inventory():
    return {"items": list(INVENTORY_DB.values()), "total": len(INVENTORY_DB)}

@app.get("/api/v1/inventory/{sku}")
async def get_item(sku: str):
    if sku not in INVENTORY_DB:
        raise HTTPException(status_code=404, detail=f"SKU {sku} not found")
    return INVENTORY_DB[sku]
EOF
```

4. Crear el código del **users-service**:

```bash
cat > /home/student/msvc-labs/services/users-service/main.py << 'EOF'
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel

app = FastAPI(title="Users Service", version="1.0.0")

class User(BaseModel):
    id: int
    username: str
    email: str
    role: str

USERS_DB = {
    1: User(id=1, username="admin-user", email="admin@example.com", role="admin"),
    2: User(id=2, username="readonly-user", email="reader@example.com", role="viewer"),
}

@app.get("/health")
async def health():
    return {"status": "healthy", "service": "users-service", "version": "1.0.0"}

@app.get("/api/v1/users")
async def list_users():
    return {"users": list(USERS_DB.values()), "total": len(USERS_DB)}

@app.get("/api/v1/users/{user_id}")
async def get_user(user_id: int):
    if user_id not in USERS_DB:
        raise HTTPException(status_code=404, detail=f"User {user_id} not found")
    return USERS_DB[user_id]
EOF
```

5. Crear el **Dockerfile** compartido:

```bash
cat > /home/student/msvc-labs/docker/Dockerfile << 'EOF'
FROM python:3.12.3-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY main.py .

EXPOSE 8000

CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
EOF
```

6. Configurar Docker para usar el daemon de minikube y construir las imágenes:

```bash
eval $(minikube docker-env --profile=msvc-gateway)

# Construir orders-service
docker build \
  -f /home/student/msvc-labs/docker/Dockerfile \
  --build-context requirements=/home/student/msvc-labs/docker \
  -t orders-service:1.0.0 \
  /home/student/msvc-labs/services/orders-service/

# Construir inventory-service
docker build \
  -f /home/student/msvc-labs/docker/Dockerfile \
  -t inventory-service:1.0.0 \
  /home/student/msvc-labs/services/inventory-service/

# Construir users-service
docker build \
  -f /home/student/msvc-labs/docker/Dockerfile \
  -t users-service:1.0.0 \
  /home/student/msvc-labs/services/users-service/
```

> **Nota:** Como el Dockerfile referencia `requirements.txt` en el mismo directorio, copie el archivo de requisitos junto al `main.py` de cada servicio antes de construir:

```bash
cp /home/student/msvc-labs/docker/requirements.txt /home/student/msvc-labs/services/orders-service/
cp /home/student/msvc-labs/docker/requirements.txt /home/student/msvc-labs/services/inventory-service/
cp /home/student/msvc-labs/docker/requirements.txt /home/student/msvc-labs/services/users-service/

# Reconstruir con contexto correcto
docker build -f /home/student/msvc-labs/docker/Dockerfile -t orders-service:1.0.0 /home/student/msvc-labs/services/orders-service/
docker build -f /home/student/msvc-labs/docker/Dockerfile -t inventory-service:1.0.0 /home/student/msvc-labs/services/inventory-service/
docker build -f /home/student/msvc-labs/docker/Dockerfile -t users-service:1.0.0 /home/student/msvc-labs/services/users-service/
```

### Salida Esperada

```
Successfully built <hash>
Successfully tagged orders-service:1.0.0
Successfully built <hash>
Successfully tagged inventory-service:1.0.0
Successfully built <hash>
Successfully tagged users-service:1.0.0
```

### Verificación

```bash
docker images | grep -E "orders-service|inventory-service|users-service"
```

Debe mostrar las tres imágenes con tag `1.0.0`.

---

## Paso 3: Desplegar los microservicios en Kubernetes

### Objetivo

Crear los Deployments y Services ClusterIP para los tres microservicios en el namespace `microservices`.

### Instrucciones

1. Crear el manifiesto de Deployment y Service para **orders-service**:

```bash
cat > /home/student/msvc-labs/k8s/deployments/orders-service.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: orders-service
  namespace: microservices
  labels:
    app: orders-service
    version: "1.0.0"
spec:
  replicas: 2
  selector:
    matchLabels:
      app: orders-service
  template:
    metadata:
      labels:
        app: orders-service
        version: "1.0.0"
    spec:
      containers:
      - name: orders-service
        image: orders-service:1.0.0
        imagePullPolicy: Never
        ports:
        - containerPort: 8000
          name: http
        livenessProbe:
          httpGet:
            path: /health
            port: 8000
          initialDelaySeconds: 5
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /health
            port: 8000
          initialDelaySeconds: 3
          periodSeconds: 5
        resources:
          requests:
            memory: "128Mi"
            cpu: "100m"
          limits:
            memory: "256Mi"
            cpu: "250m"
EOF
```

2. Crear el Service para **orders-service**:

```bash
cat > /home/student/msvc-labs/k8s/services/orders-service.yaml << 'EOF'
apiVersion: v1
kind: Service
metadata:
  name: orders-service
  namespace: microservices
  labels:
    app: orders-service
spec:
  type: ClusterIP
  selector:
    app: orders-service
  ports:
  - port: 8000
    targetPort: 8000
    protocol: TCP
    name: http
EOF
```

3. Crear Deployment y Service para **inventory-service**:

```bash
cat > /home/student/msvc-labs/k8s/deployments/inventory-service.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: inventory-service
  namespace: microservices
  labels:
    app: inventory-service
    version: "1.0.0"
spec:
  replicas: 2
  selector:
    matchLabels:
      app: inventory-service
  template:
    metadata:
      labels:
        app: inventory-service
        version: "1.0.0"
    spec:
      containers:
      - name: inventory-service
        image: inventory-service:1.0.0
        imagePullPolicy: Never
        ports:
        - containerPort: 8000
          name: http
        livenessProbe:
          httpGet:
            path: /health
            port: 8000
          initialDelaySeconds: 5
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /health
            port: 8000
          initialDelaySeconds: 3
          periodSeconds: 5
        resources:
          requests:
            memory: "128Mi"
            cpu: "100m"
          limits:
            memory: "256Mi"
            cpu: "250m"
EOF

cat > /home/student/msvc-labs/k8s/services/inventory-service.yaml << 'EOF'
apiVersion: v1
kind: Service
metadata:
  name: inventory-service
  namespace: microservices
  labels:
    app: inventory-service
spec:
  type: ClusterIP
  selector:
    app: inventory-service
  ports:
  - port: 8000
    targetPort: 8000
    protocol: TCP
    name: http
EOF
```

4. Crear Deployment y Service para **users-service**:

```bash
cat > /home/student/msvc-labs/k8s/deployments/users-service.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: users-service
  namespace: microservices
  labels:
    app: users-service
    version: "1.0.0"
spec:
  replicas: 2
  selector:
    matchLabels:
      app: users-service
  template:
    metadata:
      labels:
        app: users-service
        version: "1.0.0"
    spec:
      containers:
      - name: users-service
        image: users-service:1.0.0
        imagePullPolicy: Never
        ports:
        - containerPort: 8000
          name: http
        livenessProbe:
          httpGet:
            path: /health
            port: 8000
          initialDelaySeconds: 5
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /health
            port: 8000
          initialDelaySeconds: 3
          periodSeconds: 5
        resources:
          requests:
            memory: "128Mi"
            cpu: "100m"
          limits:
            memory: "256Mi"
            cpu: "250m"
EOF

cat > /home/student/msvc-labs/k8s/services/users-service.yaml << 'EOF'
apiVersion: v1
kind: Service
metadata:
  name: users-service
  namespace: microservices
  labels:
    app: users-service
spec:
  type: ClusterIP
  selector:
    app: users-service
  ports:
  - port: 8000
    targetPort: 8000
    protocol: TCP
    name: http
EOF
```

5. Aplicar todos los manifiestos:

```bash
kubectl apply -f /home/student/msvc-labs/k8s/deployments/
kubectl apply -f /home/student/msvc-labs/k8s/services/
```

6. Esperar a que todos los pods estén en estado Ready:

```bash
kubectl wait --for=condition=ready pod -l app=orders-service -n microservices --timeout=120s
kubectl wait --for=condition=ready pod -l app=inventory-service -n microservices --timeout=120s
kubectl wait --for=condition=ready pod -l app=users-service -n microservices --timeout=120s
```

### Salida Esperada

```
deployment.apps/orders-service created
deployment.apps/inventory-service created
deployment.apps/users-service created
service/orders-service created
service/inventory-service created
service/users-service created
pod/orders-service-xxxxx condition met
pod/inventory-service-xxxxx condition met
pod/users-service-xxxxx condition met
```

### Verificación

```bash
kubectl get pods -n microservices -o wide
kubectl get svc -n microservices
```

Debe mostrar 6 pods (2 por servicio) en estado `Running` y 3 servicios ClusterIP.

---

## Paso 4: Instalar Traefik 3.0.4 con Helm

### Objetivo

Desplegar Traefik como Ingress Controller en el namespace `traefik` usando Helm, habilitando CRDs de IngressRoute y el dashboard en puerto 9000.

### Instrucciones

1. Agregar el repositorio de Helm de Traefik:

```bash
helm repo add traefik https://traefik.github.io/charts
helm repo update
```

2. Crear el archivo de valores personalizados para Traefik:

```bash
cat > /home/student/msvc-labs/k8s/traefik/values.yaml << 'EOF'
image:
  tag: "3.0.4"

deployment:
  replicas: 1

ingressRoute:
  dashboard:
    enabled: true

ports:
  web:
    port: 8000
    expose:
      default: true
    exposedPort: 80
    protocol: TCP
  websecure:
    port: 8443
    expose:
      default: true
    exposedPort: 443
    protocol: TCP
  traefik:
    port: 9000
    expose:
      default: true
    exposedPort: 9000
    protocol: TCP

service:
  type: NodePort

providers:
  kubernetesCRD:
    enabled: true
    allowCrossNamespace: true
    namespaces: []
  kubernetesIngress:
    enabled: false

additionalArguments:
  - "--api.insecure=true"
  - "--api.dashboard=true"
  - "--log.level=INFO"

resources:
  requests:
    cpu: "100m"
    memory: "128Mi"
  limits:
    cpu: "500m"
    memory: "256Mi"
EOF
```

3. Instalar Traefik con Helm:

```bash
helm install traefik traefik/traefik \
  --namespace traefik \
  --values /home/student/msvc-labs/k8s/traefik/values.yaml \
  --version 28.3.0 \
  --wait --timeout 120s
```

> **Nota:** La versión del chart Helm 28.3.0 empaqueta Traefik 3.0.4. Verifique con `helm search repo traefik/traefik --versions | grep 28.3` si necesita confirmar.

4. Verificar que Traefik está corriendo:

```bash
kubectl get pods -n traefik
kubectl get svc -n traefik
```

### Salida Esperada

```
NAME                       READY   STATUS    RESTARTS   AGE
traefik-xxxxxxxxxx-xxxxx   1/1     Running   0          30s

NAME      TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)                                     AGE
traefik   NodePort   10.96.xxx.xxx   <none>        80:3xxxx/TCP,443:3xxxx/TCP,9000:3xxxx/TCP   30s
```

### Verificación

```bash
# Obtener la URL del dashboard de Traefik
TRAEFIK_URL=$(minikube service traefik -n traefik --url --profile=msvc-gateway | grep 9000)
echo "Dashboard URL: ${TRAEFIK_URL}/dashboard/"

# Verificar que el API de Traefik responde
curl -s "${TRAEFIK_URL}/api/version" | python3 -m json.tool
```

Debe mostrar la versión de Traefik (`"Version": "3.0.4"`).

---

## Paso 5: Configurar middlewares de Rate Limiting

### Objetivo

Crear middlewares de Traefik que implementen rate limiting diferenciado: 100 req/min para orders, 200 req/min para inventory y 50 req/min para users.

### Instrucciones

1. Crear el middleware de rate limiting para **orders-service** (100 req/min ≈ ~1.67 req/s):

```bash
cat > /home/student/msvc-labs/k8s/traefik/middlewares/ratelimit-orders.yaml << 'EOF'
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: ratelimit-orders
  namespace: microservices
spec:
  rateLimit:
    average: 100
    period: 1m
    burst: 20
EOF
```

2. Crear el middleware de rate limiting para **inventory-service** (200 req/min):

```bash
cat > /home/student/msvc-labs/k8s/traefik/middlewares/ratelimit-inventory.yaml << 'EOF'
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: ratelimit-inventory
  namespace: microservices
spec:
  rateLimit:
    average: 200
    period: 1m
    burst: 40
EOF
```

3. Crear el middleware de rate limiting para **users-service** (50 req/min):

```bash
cat > /home/student/msvc-labs/k8s/traefik/middlewares/ratelimit-users.yaml << 'EOF'
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: ratelimit-users
  namespace: microservices
spec:
  rateLimit:
    average: 50
    period: 1m
    burst: 10
EOF
```

4. Aplicar los middlewares:

```bash
kubectl apply -f /home/student/msvc-labs/k8s/traefik/middlewares/
```

### Salida Esperada

```
middleware.traefik.io/ratelimit-orders created
middleware.traefik.io/ratelimit-inventory created
middleware.traefik.io/ratelimit-users created
```

### Verificación

```bash
kubectl get middlewares -n microservices
```

Debe mostrar tres middlewares listados.

---

## Paso 6: Configurar middleware de Circuit Breaker

### Objetivo

Crear un middleware de circuit breaker que abra el circuito cuando el 50% de las solicitudes en una ventana de 10 segundos resulten en errores.

### Instrucciones

1. Crear el middleware de circuit breaker compartido:

```bash
cat > /home/student/msvc-labs/k8s/traefik/middlewares/circuitbreaker-default.yaml << 'EOF'
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: circuitbreaker-default
  namespace: microservices
spec:
  circuitBreaker:
    expression: "ResponseCodeRatio(500, 600, 0, 600) > 0.50"
    checkPeriod: 10s
    fallbackDuration: 15s
    recoveryDuration: 30s
EOF
```

2. Aplicar el middleware:

```bash
kubectl apply -f /home/student/msvc-labs/k8s/traefik/middlewares/circuitbreaker-default.yaml
```

### Salida Esperada

```
middleware.traefik.io/circuitbreaker-default created
```

### Verificación

```bash
kubectl get middlewares -n microservices
```

Debe mostrar 4 middlewares: `ratelimit-orders`, `ratelimit-inventory`, `ratelimit-users`, `circuitbreaker-default`.

---

## Paso 7: Crear IngressRoutes para enrutar tráfico

### Objetivo

Configurar IngressRoute CRDs que enruten el tráfico desde el dominio `microservices.local` hacia cada microservicio, aplicando los middlewares de rate limiting y circuit breaker.

### Instrucciones

1. Crear la IngressRoute para **orders-service**:

```bash
cat > /home/student/msvc-labs/k8s/traefik/ingress-routes/orders-route.yaml << 'EOF'
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: orders-route
  namespace: microservices
spec:
  entryPoints:
    - web
  routes:
    - match: Host(`microservices.local`) && PathPrefix(`/api/v1/orders`)
      kind: Rule
      services:
        - name: orders-service
          port: 8000
      middlewares:
        - name: ratelimit-orders
        - name: circuitbreaker-default
EOF
```

2. Crear la IngressRoute para **inventory-service**:

```bash
cat > /home/student/msvc-labs/k8s/traefik/ingress-routes/inventory-route.yaml << 'EOF'
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: inventory-route
  namespace: microservices
spec:
  entryPoints:
    - web
  routes:
    - match: Host(`microservices.local`) && PathPrefix(`/api/v1/inventory`)
      kind: Rule
      services:
        - name: inventory-service
          port: 8000
      middlewares:
        - name: ratelimit-inventory
        - name: circuitbreaker-default
EOF
```

3. Crear la IngressRoute para **users-service**:

```bash
cat > /home/student/msvc-labs/k8s/traefik/ingress-routes/users-route.yaml << 'EOF'
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: users-route
  namespace: microservices
spec:
  entryPoints:
    - web
  routes:
    - match: Host(`microservices.local`) && PathPrefix(`/api/v1/users`)
      kind: Rule
      services:
        - name: users-service
          port: 8000
      middlewares:
        - name: ratelimit-users
        - name: circuitbreaker-default
EOF
```

4. Aplicar las IngressRoutes:

```bash
kubectl apply -f /home/student/msvc-labs/k8s/traefik/ingress-routes/
```

5. Configurar la resolución DNS local para `microservices.local`:

```bash
MINIKUBE_IP=$(minikube ip --profile=msvc-gateway)
echo "${MINIKUBE_IP} microservices.local" | sudo tee -a /etc/hosts
```

6. Obtener el puerto NodePort del entrypoint web de Traefik:

```bash
TRAEFIK_WEB_PORT=$(kubectl get svc traefik -n traefik -o jsonpath='{.spec.ports[?(@.name=="web")].nodePort}')
echo "Traefik Web Port: ${TRAEFIK_WEB_PORT}"
```

### Salida Esperada

```
ingressroute.traefik.io/orders-route created
ingressroute.traefik.io/inventory-route created
ingressroute.traefik.io/users-route created
```

### Verificación

```bash
kubectl get ingressroutes -n microservices
```

Debe mostrar tres IngressRoutes. Prueba rápida de conectividad:

```bash
curl -s -H "Host: microservices.local" http://${MINIKUBE_IP}:${TRAEFIK_WEB_PORT}/api/v1/orders | python3 -m json.tool
```

Debe retornar la lista de órdenes.

---

## Paso 8: Validar el enrutamiento básico

### Objetivo

Confirmar que Traefik enruta correctamente las solicitudes a cada microservicio según el prefijo de ruta.

### Instrucciones

1. Probar el endpoint de **orders-service**:

```bash
curl -s -H "Host: microservices.local" \
  http://${MINIKUBE_IP}:${TRAEFIK_WEB_PORT}/api/v1/orders | python3 -m json.tool
```

2. Probar el endpoint de **inventory-service**:

```bash
curl -s -H "Host: microservices.local" \
  http://${MINIKUBE_IP}:${TRAEFIK_WEB_PORT}/api/v1/inventory | python3 -m json.tool
```

3. Probar el endpoint de **users-service**:

```bash
curl -s -H "Host: microservices.local" \
  http://${MINIKUBE_IP}:${TRAEFIK_WEB_PORT}/api/v1/users | python3 -m json.tool
```

4. Probar un endpoint específico con parámetro:

```bash
curl -s -H "Host: microservices.local" \
  http://${MINIKUBE_IP}:${TRAEFIK_WEB_PORT}/api/v1/orders/1 | python3 -m json.tool
```

5. Verificar que una ruta inexistente retorna 404:

```bash
curl -s -o /dev/null -w "%{http_code}" -H "Host: microservices.local" \
  http://${MINIKUBE_IP}:${TRAEFIK_WEB_PORT}/api/v1/nonexistent
```

### Salida Esperada

```json
{
    "orders": [
        {"id": 1, "product": "Laptop Pro", "quantity": 1, "status": "confirmed", "created_at": "2024-06-01T10:00:00Z"},
        ...
    ],
    "total": 3
}
```

```json
{
    "items": [
        {"sku": "SKU-001", "name": "Laptop Pro", "quantity": 42, "warehouse": "Madrid"},
        ...
    ],
    "total": 3
}
```

```json
{
    "users": [
        {"id": 1, "username": "admin-user", "email": "admin@example.com", "role": "admin"},
        ...
    ],
    "total": 2
}
```

El código HTTP para la ruta inexistente debe ser `404`.

### Verificación

Los tres servicios responden correctamente a través del gateway con datos válidos.

---

## Paso 9: Validar Rate Limiting con prueba de carga

### Objetivo

Verificar que el middleware de rate limiting rechaza solicitudes que excedan los límites configurados (50 req/min para users-service).

### Instrucciones

1. Crear el script de prueba de rate limiting:

```bash
cat > /home/student/msvc-labs/scripts/test_ratelimit.py << 'EOF'
#!/usr/bin/env python3
"""
Script para validar rate limiting en Traefik.
Envía ráfagas de solicitudes al users-service (límite: 50 req/min, burst: 10)
y cuenta las respuestas 429 (Too Many Requests).
"""

import subprocess
import sys
import time
import json

MINIKUBE_IP = subprocess.check_output(
    ["minikube", "ip", "--profile=msvc-gateway"]
).decode().strip()

TRAEFIK_PORT = subprocess.check_output(
    ["kubectl", "get", "svc", "traefik", "-n", "traefik",
     "-o", "jsonpath={.spec.ports[?(@.name==\"web\")].nodePort}"]
).decode().strip()

BASE_URL = f"http://{MINIKUBE_IP}:{TRAEFIK_PORT}"
ENDPOINT = "/api/v1/users"
TOTAL_REQUESTS = 80  # Más que el límite de 50/min + burst 10

print(f"{'='*60}")
print(f"Prueba de Rate Limiting - users-service")
print(f"Límite configurado: 50 req/min, burst: 10")
print(f"Solicitudes a enviar: {TOTAL_REQUESTS}")
print(f"URL: {BASE_URL}{ENDPOINT}")
print(f"{'='*60}\n")

results = {"200": 0, "429": 0, "other": 0}

for i in range(TOTAL_REQUESTS):
    try:
        result = subprocess.run(
            ["curl", "-s", "-o", "/dev/null", "-w", "%{http_code}",
             "-H", "Host: microservices.local",
             f"{BASE_URL}{ENDPOINT}"],
            capture_output=True, text=True, timeout=5
        )
        code = result.stdout.strip()
        if code == "200":
            results["200"] += 1
        elif code == "429":
            results["429"] += 1
        else:
            results["other"] += 1

        if (i + 1) % 20 == 0:
            print(f"  Progreso: {i+1}/{TOTAL_REQUESTS} solicitudes enviadas...")

    except Exception as e:
        results["other"] += 1

print(f"\n{'='*60}")
print(f"RESULTADOS:")
print(f"  ✅ Respuestas 200 (OK):              {results['200']}")
print(f"  🚫 Respuestas 429 (Rate Limited):    {results['429']}")
print(f"  ⚠️  Otras respuestas:                 {results['other']}")
print(f"{'='*60}")

if results["429"] > 0:
    print("\n✅ PRUEBA EXITOSA: Rate limiting está funcionando correctamente.")
    print(f"   Se bloquearon {results['429']} solicitudes que excedieron el límite.")
else:
    print("\n⚠️  ATENCIÓN: No se detectaron respuestas 429.")
    print("   El rate limiting podría no estar activo o el burst es suficiente.")
    sys.exit(1)
EOF

chmod +x /home/student/msvc-labs/scripts/test_ratelimit.py
```

2. Ejecutar la prueba:

```bash
python3 /home/student/msvc-labs/scripts/test_ratelimit.py
```

### Salida Esperada

```
============================================================
Prueba de Rate Limiting - users-service
Límite configurado: 50 req/min, burst: 10
Solicitudes a enviar: 80
URL: http://192.168.49.2:3xxxx/api/v1/users
============================================================

  Progreso: 20/80 solicitudes enviadas...
  Progreso: 40/80 solicitudes enviadas...
  Progreso: 60/80 solicitudes enviadas...
  Progreso: 80/80 solicitudes enviadas...

============================================================
RESULTADOS:
  ✅ Respuestas 200 (OK):              58
  🚫 Respuestas 429 (Rate Limited):    22
  ⚠️  Otras respuestas:                 0
============================================================

✅ PRUEBA EXITOSA: Rate limiting está funcionando correctamente.
   Se bloquearon 22 solicitudes que excedieron el límite.
```

### Verificación

El número de respuestas 429 debe ser mayor que 0, confirmando que el rate limiting está activo. Los valores exactos pueden variar según la velocidad de ejecución.

---

## Paso 10: Validar Circuit Breaker con simulación de fallos

### Objetivo

Verificar que el circuit breaker de Traefik abre el circuito cuando más del 50% de las solicitudes generan errores 5xx en la ventana de 10 segundos.

### Instrucciones

1. Crear el script de prueba del circuit breaker:

```bash
cat > /home/student/msvc-labs/scripts/test_circuitbreaker.py << 'EOF'
#!/usr/bin/env python3
"""
Script para validar circuit breaker en Traefik.
Envía solicitudes al endpoint de fallo simulado de orders-service
para disparar la apertura del circuito.
"""

import subprocess
import time
import sys

MINIKUBE_IP = subprocess.check_output(
    ["minikube", "ip", "--profile=msvc-gateway"]
).decode().strip()

TRAEFIK_PORT = subprocess.check_output(
    ["kubectl", "get", "svc", "traefik", "-n", "traefik",
     "-o", "jsonpath={.spec.ports[?(@.name==\"web\")].nodePort}"]
).decode().strip()

BASE_URL = f"http://{MINIKUBE_IP}:{TRAEFIK_PORT}"
FAIL_ENDPOINT = "/api/v1/orders/fail/simulate"
HEALTHY_ENDPOINT = "/api/v1/orders"

print(f"{'='*60}")
print(f"Prueba de Circuit Breaker - orders-service")
print(f"Umbral: 50% errores en ventana de 10s")
print(f"{'='*60}\n")

# Fase 1: Generar errores para abrir el circuito
print("📍 Fase 1: Generando errores 500 para abrir el circuito...")
error_count = 0
for i in range(30):
    result = subprocess.run(
        ["curl", "-s", "-o", "/dev/null", "-w", "%{http_code}",
         "-H", "Host: microservices.local",
         f"{BASE_URL}{FAIL_ENDPOINT}"],
        capture_output=True, text=True, timeout=5
    )
    code = result.stdout.strip()
    if code == "500":
        error_count += 1
    time.sleep(0.1)

print(f"   Errores 500 generados: {error_count}/30")

# Fase 2: Verificar que el circuito está abierto
print("\n📍 Fase 2: Verificando estado del circuito (solicitudes al endpoint saludable)...")
time.sleep(1)

circuit_open_responses = 0
for i in range(10):
    result = subprocess.run(
        ["curl", "-s", "-o", "/dev/null", "-w", "%{http_code}",
         "-H", "Host: microservices.local",
         f"{BASE_URL}{HEALTHY_ENDPOINT}"],
        capture_output=True, text=True, timeout=5
    )
    code = result.stdout.strip()
    if code == "503":
        circuit_open_responses += 1
    time.sleep(0.2)

print(f"   Respuestas 503 (circuito abierto): {circuit_open_responses}/10")

# Fase 3: Esperar recuperación
print(f"\n📍 Fase 3: Esperando recuperación del circuito (30s)...")
time.sleep(35)

recovery_success = 0
for i in range(10):
    result = subprocess.run(
        ["curl", "-s", "-o", "/dev/null", "-w", "%{http_code}",
         "-H", "Host: microservices.local",
         f"{BASE_URL}{HEALTHY_ENDPOINT}"],
        capture_output=True, text=True, timeout=5
    )
    code = result.stdout.strip()
    if code == "200":
        recovery_success += 1
    time.sleep(0.2)

print(f"   Respuestas 200 tras recuperación: {recovery_success}/10")

print(f"\n{'='*60}")
print("RESULTADOS:")
if circuit_open_responses > 0:
    print(f"  ✅ Circuit breaker se ACTIVÓ: {circuit_open_responses} solicitudes bloqueadas con 503")
else:
    print(f"  ⚠️  Circuit breaker NO se activó visiblemente.")
    print(f"      Esto puede ocurrir si la expresión evalúa solo el ratio del servicio específico.")

if recovery_success > 5:
    print(f"  ✅ Servicio se RECUPERÓ: {recovery_success}/10 solicitudes exitosas")
else:
    print(f"  ⚠️  Recuperación parcial: {recovery_success}/10 solicitudes exitosas")
print(f"{'='*60}")
EOF

chmod +x /home/student/msvc-labs/scripts/test_circuitbreaker.py
```

2. Ejecutar la prueba del circuit breaker:

```bash
python3 /home/student/msvc-labs/scripts/test_circuitbreaker.py
```

### Salida Esperada

```
============================================================
Prueba de Circuit Breaker - orders-service
Umbral: 50% errores en ventana de 10s
============================================================

📍 Fase 1: Generando errores 500 para abrir el circuito...
   Errores 500 generados: 30/30

📍 Fase 2: Verificando estado del circuito (solicitudes al endpoint saludable)...
   Respuestas 503 (circuito abierto): 7/10

📍 Fase 3: Esperando recuperación del circuito (30s)...
   Respuestas 200 tras recuperación: 10/10

============================================================
RESULTADOS:
  ✅ Circuit breaker se ACTIVÓ: 7 solicitudes bloqueadas con 503
  ✅ Servicio se RECUPERÓ: 10/10 solicitudes exitosas
============================================================
```

### Verificación

Las respuestas 503 en la Fase 2 confirman que el circuit breaker se activó. La recuperación en la Fase 3 confirma que el circuito se cierra tras el período de recuperación configurado (30s).

---

## Validación y Pruebas Finales

Ejecute la siguiente secuencia de validación completa para confirmar que todo el sistema funciona correctamente:

```bash
cat > /home/student/msvc-labs/scripts/validate_all.sh << 'EOF'
#!/bin/bash
set -e

echo "=========================================="
echo "VALIDACIÓN COMPLETA DEL LABORATORIO"
echo "=========================================="

MINIKUBE_IP=$(minikube ip --profile=msvc-gateway)
TRAEFIK_PORT=$(kubectl get svc traefik -n traefik -o jsonpath='{.spec.ports[?(@.name=="web")].nodePort}')
BASE="http://${MINIKUBE_IP}:${TRAEFIK_PORT}"
PASS=0
FAIL=0

check() {
    local desc="$1"
    local expected="$2"
    local actual="$3"
    if [ "$actual" == "$expected" ]; then
        echo "  ✅ ${desc}"
        ((PASS++))
    else
        echo "  ❌ ${desc} (esperado: ${expected}, obtenido: ${actual})"
        ((FAIL++))
    fi
}

echo ""
echo "1. Verificando pods en ejecución..."
PODS_READY=$(kubectl get pods -n microservices --field-selector=status.phase=Running --no-headers | wc -l)
check "Pods running en microservices (esperado: 6)" "6" "${PODS_READY}"

echo ""
echo "2. Verificando Traefik..."
TRAEFIK_PODS=$(kubectl get pods -n traefik --field-selector=status.phase=Running --no-headers | wc -l)
check "Traefik pod running" "1" "${TRAEFIK_PODS}"

echo ""
echo "3. Verificando enrutamiento..."
CODE_ORDERS=$(curl -s -o /dev/null -w "%{http_code}" -H "Host: microservices.local" "${BASE}/api/v1/orders")
check "Orders endpoint accesible" "200" "${CODE_ORDERS}"

CODE_INVENTORY=$(curl -s -o /dev/null -w "%{http_code}" -H "Host: microservices.local" "${BASE}/api/v1/inventory")
check "Inventory endpoint accesible" "200" "${CODE_INVENTORY}"

CODE_USERS=$(curl -s -o /dev/null -w "%{http_code}" -H "Host: microservices.local" "${BASE}/api/v1/users")
check "Users endpoint accesible" "200" "${CODE_USERS}"

echo ""
echo "4. Verificando middlewares..."
MW_COUNT=$(kubectl get middlewares -n microservices --no-headers | wc -l)
check "Middlewares configurados (esperado: 4)" "4" "${MW_COUNT}"

echo ""
echo "5. Verificando IngressRoutes..."
IR_COUNT=$(kubectl get ingressroutes -n microservices --no-headers | wc -l)
check "IngressRoutes configuradas (esperado: 3)" "3" "${IR_COUNT}"

echo ""
echo "6. Verificando dashboard Traefik..."
DASHBOARD_PORT=$(kubectl get svc traefik -n traefik -o jsonpath='{.spec.ports[?(@.name=="traefik")].nodePort}')
CODE_DASHBOARD=$(curl -s -o /dev/null -w "%{http_code}" "http://${MINIKUBE_IP}:${DASHBOARD_PORT}/api/version")
check "Dashboard Traefik accesible" "200" "${CODE_DASHBOARD}"

echo ""
echo "=========================================="
echo "RESUMEN: ${PASS} pruebas exitosas, ${FAIL} fallidas"
echo "=========================================="

if [ ${FAIL} -gt 0 ]; then
    exit 1
fi
EOF

chmod +x /home/student/msvc-labs/scripts/validate_all.sh
bash /home/student/msvc-labs/scripts/validate_all.sh
```

### Resultado Esperado

```
==========================================
VALIDACIÓN COMPLETA DEL LABORATORIO
==========================================

1. Verificando pods en ejecución...
  ✅ Pods running en microservices (esperado: 6)

2. Verificando Traefik...
  ✅ Traefik pod running

3. Verificando enrutamiento...
  ✅ Orders endpoint accesible
  ✅ Inventory endpoint accesible
  ✅ Users endpoint accesible

4. Verificando middlewares...
  ✅ Middlewares configurados (esperado: 4)

5. Verificando IngressRoutes...
  ✅ IngressRoutes configuradas (esperado: 3)

6. Verificando dashboard Traefik...
  ✅ Dashboard Traefik accesible

==========================================
RESUMEN: 8 pruebas exitosas, 0 fallidas
==========================================
```

---

## Solución de Problemas

### Problema 1: Pods de microservicios en estado ImagePullBackOff

**Síntomas:**

```
kubectl get pods -n microservices
NAME                               READY   STATUS             RESTARTS   AGE
orders-service-xxx-yyy             0/1     ImagePullBackOff   0          60s
```

**Causa:** El Docker daemon del host no está sincronizado con el de minikube. Las imágenes se construyeron en el daemon del host pero minikube no puede acceder a ellas.

**Solución:**

```bash
# Verificar que se está usando el daemon de minikube
eval $(minikube docker-env --profile=msvc-gateway)

# Confirmar que las imágenes existen en el daemon correcto
docker images | grep -E "orders|inventory|users"

# Si no aparecen, reconstruir dentro del contexto de minikube
docker build -f /home/student/msvc-labs/docker/Dockerfile -t orders-service:1.0.0 /home/student/msvc-labs/services/orders-service/
docker build -f /home/student/msvc-labs/docker/Dockerfile -t inventory-service:1.0.0 /home/student/msvc-labs/services/inventory-service/
docker build -f /home/student/msvc-labs/docker/Dockerfile -t users-service:1.0.0 /home/student/msvc-labs/services/users-service/

# Reiniciar los deployments
kubectl rollout restart deployment -n microservices
```

Verificar que `imagePullPolicy: Never` está configurado en los Deployments para que Kubernetes use las imágenes locales.

---

### Problema 2: IngressRoute no enruta tráfico (respuesta 404 de Traefik)

**Síntomas:**

```bash
curl -H "Host: microservices.local" http://<minikube-ip>:<port>/api/v1/orders
# Responde: 404 page not found
```

**Causa:** Traefik no tiene permisos para leer CRDs en otros namespaces, o la opción `allowCrossNamespace` no está habilitada. Las IngressRoutes están en el namespace `microservices` pero Traefik está en `traefik`.

**Solución:**

```bash
# Verificar que allowCrossNamespace está habilitado
kubectl get deployment traefik -n traefik -o yaml | grep -A5 "kubernetesCRD"

# Si no está habilitado, actualizar la instalación de Helm
helm upgrade traefik traefik/traefik \
  --namespace traefik \
  --values /home/student/msvc-labs/k8s/traefik/values.yaml \
  --wait

# Verificar que Traefik detecta las rutas
DASHBOARD_PORT=$(kubectl get svc traefik -n traefik -o jsonpath='{.spec.ports[?(@.name=="traefik")].nodePort}')
MINIKUBE_IP=$(minikube ip --profile=msvc-gateway)
curl -s "http://${MINIKUBE_IP}:${DASHBOARD_PORT}/api/http/routers" | python3 -m json.tool | grep "orders"

# Verificar logs de Traefik para errores de permisos
kubectl logs -n traefik -l app.kubernetes.io/name=traefik --tail=50 | grep -i "error\|forbidden"
```

Si los logs muestran errores RBAC, verificar que el ClusterRole de Traefik incluye permisos sobre `traefik.io` resources en todos los namespaces.

---

## Limpieza

Para eliminar todos los recursos creados en este laboratorio:

```bash
# Eliminar IngressRoutes y Middlewares
kubectl delete -f /home/student/msvc-labs/k8s/traefik/ingress-routes/
kubectl delete -f /home/student/msvc-labs/k8s/traefik/middlewares/

# Eliminar microservicios
kubectl delete -f /home/student/msvc-labs/k8s/services/
kubectl delete -f /home/student/msvc-labs/k8s/deployments/

# Desinstalar Traefik
helm uninstall traefik -n traefik

# Eliminar namespaces
kubectl delete namespace microservices
kubectl delete namespace traefik

# Eliminar entrada en /etc/hosts
sudo sed -i '/microservices.local/d' /etc/hosts

# Detener minikube (opcional - mantener si continúas con labs posteriores)
# minikube stop --profile=msvc-gateway

# Eliminar clúster completamente (solo si no necesitas el entorno)
# minikube delete --profile=msvc-gateway
```

> **Nota:** Si vas a continuar con laboratorios posteriores del batch 2, **no elimines** el clúster minikube ni el directorio `/home/student/msvc-labs/`. Estos recursos se reutilizan como base.

---

## Resumen

En este laboratorio has completado las siguientes tareas:

| Tarea | Estado |
|-------|--------|
| Clúster minikube con namespaces configurados | ✅ |
| Tres microservicios FastAPI desplegados como Deployments | ✅ |
| Traefik 3.0.4 instalado via Helm con CRDs habilitados | ✅ |
| IngressRoutes configuradas para enrutamiento por prefijo de ruta | ✅ |
| Rate limiting diferenciado (100/200/50 req/min) | ✅ |
| Circuit breaker con umbral del 50% en ventana de 10s | ✅ |
| Validación funcional con scripts automatizados | ✅ |

### Conceptos Clave Aplicados

- **Proxy inverso simple**: Traefik actúa como punto único de entrada, enrutando por prefijo de ruta URL (`/api/v1/orders` → orders-service).
- **Configuración declarativa**: Todos los middlewares y rutas se definen como CRDs de Kubernetes, facilitando GitOps.
- **Rate limiting en el edge**: La protección se aplica antes de que el tráfico llegue a los servicios, preservando recursos backend.
- **Circuit breaker**: Protege contra fallos cascada abriendo el circuito cuando la tasa de errores supera el umbral.

### Recursos Adicionales

- [Documentación oficial de Traefik 3.0 - IngressRoute](https://doc.traefik.io/traefik/routing/providers/kubernetes-crd/)
- [Traefik Middleware - RateLimit](https://doc.traefik.io/traefik/middlewares/http/ratelimit/)
- [Traefik Middleware - CircuitBreaker](https://doc.traefik.io/traefik/middlewares/http/circuitbreaker/)
- [FastAPI - Deployment en Kubernetes](https://fastapi.tiangolo.com/deployment/docker/)
