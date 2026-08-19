# Pipeline Completo desde Commit hasta Despliegue en Kubernetes

## Metadata

| Campo | Valor |
|-------|-------|
| **Duración** | 66 minutos |
| **Complejidad** | Alta |
| **Nivel Bloom** | Crear |

## Descripción General

En este laboratorio diseñarás e implementarás un pipeline CI/CD multi-etapa completo que automatiza el ciclo de vida de un microservicio Python desde el commit de código hasta su despliegue en Kubernetes. Aplicarás el patrón GitOps con separación de repositorios (aplicación vs. configuración), configurarás Argo CD como operador de reconciliación continua, e implementarás quality gates obligatorios incluyendo cobertura de tests, escaneo de vulnerabilidades con Trivy y validación de manifiestos.

## Objetivos de Aprendizaje

- [ ] Diseñar un pipeline CI/CD multi-etapa con GitHub Actions que incluya build, test, escaneo de seguridad y publicación de imagen Docker
- [ ] Configurar Argo CD como operador GitOps con sincronización automática y self-healing en un clúster Kubernetes local
- [ ] Implementar quality gates obligatorios: cobertura ≥80%, cero vulnerabilidades CRITICAL y validación de manifiestos
- [ ] Establecer despliegue progresivo con rolling update, health checks y rollback automático
- [ ] Separar correctamente repositorio de aplicación del repositorio de configuración GitOps

## Prerrequisitos

### Conocimientos Requeridos

- Experiencia con GitHub Actions (jobs, steps, secrets, artifacts)
- Comprensión de manifiestos Kubernetes (Deployment, Service, ConfigMap, HPA)
- Familiaridad con Helm 3.x (values.yaml, templates, Chart.yaml)
- Python con pytest y FastAPI (laboratorios anteriores completados)

### Acceso y Herramientas

- Clúster minikube 1.33.1 operativo con addons `ingress` y `metrics-server`
- Docker Engine 26.1.3 con Docker Compose 2.27.1
- Python 3.12.4 con pytest 8.2.2
- Acceso a internet para descargar imágenes y charts

## Entorno del Laboratorio

### Software Requerido

| Herramienta | Versión | Propósito |
|-------------|---------|-----------|
| minikube | 1.33.1 | Clúster Kubernetes local |
| kubectl | 1.30.2 | Gestión de recursos K8s |
| Helm | 3.15.2 | Empaquetado de charts |
| Argo CD | 2.11.3 | Operador GitOps |
| act | 0.2.61 | Ejecución local de GitHub Actions |
| Trivy | 0.52.2 | Escaneo de vulnerabilidades |
| Docker Buildx | 0.14.1 | Construcción de imágenes |
| Docker Registry | 2.8.3 | Registro local de imágenes |

### Configuración Inicial del Entorno

```bash
# Crear directorio base de trabajo
mkdir -p ~/microservices-lab/lab-11
cd ~/microservices-lab/lab-11

# Verificar minikube con 3 nodos
minikube status
# Si no está corriendo:
minikube start --nodes=3 --cpus=4 --memory=8192 \
  --addons=ingress,metrics-server \
  --driver=docker

# Verificar kubectl
kubectl get nodes
kubectl cluster-info

# Verificar Helm
helm version --short
```

---

## Paso 1: Configurar el Registro Docker Local

**Objetivo:** Levantar un registro Docker local en `localhost:5050` para almacenar las imágenes construidas por el pipeline sin depender de registros externos.

### Instrucciones

1. Crear y ejecutar el contenedor del registro local:

```bash
# Verificar si ya existe el registro
docker ps -a --filter name=local-registry

# Crear el registro local si no existe
docker run -d \
  --name local-registry \
  --restart=always \
  -p 5050:5000 \
  registry:2.8.3
```

2. Configurar minikube para acceder al registro local:

```bash
# Obtener la IP del host desde la perspectiva de minikube
MINIKUBE_HOST_IP=$(minikube ssh "route -n | grep '^0.0.0.0' | awk '{print \$2}'" 2>/dev/null || echo "192.168.49.1")
echo "Host IP para minikube: $MINIKUBE_HOST_IP"

# Configurar minikube para usar el registro inseguro
minikube stop
minikube start --nodes=3 --cpus=4 --memory=8192 \
  --addons=ingress,metrics-server \
  --insecure-registry="${MINIKUBE_HOST_IP}:5050" \
  --driver=docker
```

3. Verificar conectividad al registro:

```bash
# Probar push de una imagen de test
docker pull busybox:latest
docker tag busybox:latest localhost:5050/test-image:v1
docker push localhost:5050/test-image:v1

# Verificar catálogo del registro
curl -s http://localhost:5050/v2/_catalog
```

### Salida Esperada

```json
{"repositories":["test-image"]}
```

### Verificación

```bash
# El registro debe responder correctamente
curl -sf http://localhost:5050/v2/ && echo "✓ Registro operativo" || echo "✗ Registro no disponible"
```

---

## Paso 2: Crear el Repositorio de Aplicación (ms-app-repo)

**Objetivo:** Estructurar el repositorio de código fuente con el microservicio order-service, tests, Dockerfile multi-stage y workflow de GitHub Actions.

### Instrucciones

1. Inicializar el repositorio de aplicación:

```bash
cd ~/microservices-lab/lab-11
mkdir -p ms-app-repo
cd ms-app-repo
git init
git checkout -b main
```

2. Crear la estructura del proyecto:

```bash
mkdir -p src/order_service tests .github/workflows
```

3. Crear el código fuente del microservicio (`src/order_service/main.py`):

```python
# src/order_service/main.py
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from typing import Optional
from datetime import datetime
import uuid

app = FastAPI(
    title="Order Service",
    version="1.0.0",
    description="Microservicio de gestión de pedidos"
)

class OrderCreate(BaseModel):
    product_id: str
    quantity: int
    customer_id: str

class OrderResponse(BaseModel):
    order_id: str
    product_id: str
    quantity: int
    customer_id: str
    status: str
    created_at: str

# Almacenamiento en memoria para el laboratorio
orders_db: dict = {}

@app.get("/health")
def health_check():
    return {"status": "healthy", "timestamp": datetime.utcnow().isoformat()}

@app.get("/ready")
def readiness_check():
    return {"status": "ready", "service": "order-service"}

@app.post("/api/v1/orders", response_model=OrderResponse, status_code=201)
def create_order(order: OrderCreate):
    order_id = str(uuid.uuid4())
    new_order = OrderResponse(
        order_id=order_id,
        product_id=order.product_id,
        quantity=order.quantity,
        customer_id=order.customer_id,
        status="pending",
        created_at=datetime.utcnow().isoformat()
    )
    orders_db[order_id] = new_order
    return new_order

@app.get("/api/v1/orders/{order_id}", response_model=OrderResponse)
def get_order(order_id: str):
    if order_id not in orders_db:
        raise HTTPException(status_code=404, detail="Order not found")
    return orders_db[order_id]

@app.get("/api/v1/orders")
def list_orders(limit: Optional[int] = 10):
    return list(orders_db.values())[:limit]
```

4. Crear el archivo `src/order_service/__init__.py`:

```python
# src/order_service/__init__.py
```

5. Crear los tests (`tests/test_orders.py`):

```python
# tests/test_orders.py
import pytest
from fastapi.testclient import TestClient
from src.order_service.main import app, orders_db

@pytest.fixture(autouse=True)
def clear_db():
    orders_db.clear()
    yield
    orders_db.clear()

@pytest.fixture
def client():
    return TestClient(app)

class TestHealthEndpoints:
    def test_health_check(self, client):
        response = client.get("/health")
        assert response.status_code == 200
        data = response.json()
        assert data["status"] == "healthy"
        assert "timestamp" in data

    def test_readiness_check(self, client):
        response = client.get("/ready")
        assert response.status_code == 200
        assert response.json()["status"] == "ready"

class TestOrderCreation:
    def test_create_order_success(self, client):
        payload = {
            "product_id": "PROD-001",
            "quantity": 3,
            "customer_id": "CUST-100"
        }
        response = client.post("/api/v1/orders", json=payload)
        assert response.status_code == 201
        data = response.json()
        assert data["product_id"] == "PROD-001"
        assert data["quantity"] == 3
        assert data["status"] == "pending"
        assert "order_id" in data

    def test_create_order_invalid_payload(self, client):
        response = client.post("/api/v1/orders", json={"product_id": "X"})
        assert response.status_code == 422

class TestOrderRetrieval:
    def test_get_order_not_found(self, client):
        response = client.get("/api/v1/orders/nonexistent-id")
        assert response.status_code == 404

    def test_get_order_success(self, client):
        # Crear primero
        payload = {
            "product_id": "PROD-002",
            "quantity": 1,
            "customer_id": "CUST-200"
        }
        create_resp = client.post("/api/v1/orders", json=payload)
        order_id = create_resp.json()["order_id"]

        # Recuperar
        response = client.get(f"/api/v1/orders/{order_id}")
        assert response.status_code == 200
        assert response.json()["order_id"] == order_id

    def test_list_orders_empty(self, client):
        response = client.get("/api/v1/orders")
        assert response.status_code == 200
        assert response.json() == []

    def test_list_orders_with_limit(self, client):
        for i in range(5):
            client.post("/api/v1/orders", json={
                "product_id": f"PROD-{i}",
                "quantity": 1,
                "customer_id": "CUST-300"
            })
        response = client.get("/api/v1/orders?limit=3")
        assert response.status_code == 200
        assert len(response.json()) == 3
```

6. Crear `requirements.txt`:

```text
fastapi==0.111.0
uvicorn==0.29.0
pydantic==2.7.1
```

7. Crear `requirements-dev.txt`:

```text
pytest==8.2.2
pytest-cov==5.0.0
httpx==0.27.0
```

8. Crear el `Dockerfile` multi-stage:

```dockerfile
# Dockerfile
# Etapa 1: Builder
FROM python:3.12.4-slim AS builder

WORKDIR /build
COPY requirements.txt .
RUN pip install --no-cache-dir --prefix=/install -r requirements.txt

# Etapa 2: Runtime
FROM python:3.12.4-slim AS runtime

LABEL maintainer="mslab-team"
LABEL version="1.0.0"

# Crear usuario no-root
RUN groupadd -r appuser && useradd -r -g appuser appuser

WORKDIR /app

# Copiar dependencias del builder
COPY --from=builder /install /usr/local

# Copiar código fuente
COPY src/ ./src/

# Cambiar a usuario no-root
USER appuser

EXPOSE 8001

HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 \
  CMD python -c "import urllib.request; urllib.request.urlopen('http://localhost:8001/health')" || exit 1

CMD ["uvicorn", "src.order_service.main:app", "--host", "0.0.0.0", "--port", "8001"]
```

9. Crear `pytest.ini`:

```ini
[pytest]
testpaths = tests
addopts = --strict-markers -v
```

10. Realizar commit inicial:

```bash
git add -A
git commit -m "feat: initial order-service with tests and Dockerfile"
```

### Salida Esperada

```
[main (root-commit) xxxxxxx] feat: initial order-service with tests and Dockerfile
 8 files changed, ...
```

### Verificación

```bash
# Ejecutar tests localmente para confirmar que pasan
cd ~/microservices-lab/lab-11/ms-app-repo
pip install -r requirements.txt -r requirements-dev.txt -q
pytest tests/ --cov=src --cov-report=term-missing
```

La cobertura debe ser ≥80% y todos los tests deben pasar.

---

## Paso 3: Crear el Repositorio GitOps (ms-gitops-repo)

**Objetivo:** Crear el repositorio de configuración con un Helm chart que define el estado deseado del despliegue en Kubernetes, separado del código de aplicación.

### Instrucciones

1. Inicializar el repositorio GitOps:

```bash
cd ~/microservices-lab/lab-11
mkdir -p ms-gitops-repo
cd ms-gitops-repo
git init
git checkout -b main
```

2. Crear la estructura del Helm chart:

```bash
mkdir -p charts/order-service/templates
```

3. Crear `charts/order-service/Chart.yaml`:

```yaml
apiVersion: v2
name: order-service
description: Helm chart para Order Service - Pipeline CI/CD Lab
type: application
version: 0.1.0
appVersion: "1.0.0"
```

4. Crear `charts/order-service/values.yaml`:

```yaml
# values.yaml - Fuente de verdad para el estado deseado
replicaCount: 2

image:
  repository: localhost:5050/order-service
  tag: "latest"
  pullPolicy: Always

service:
  type: ClusterIP
  port: 8001

resources:
  requests:
    memory: "128Mi"
    cpu: "100m"
  limits:
    memory: "256Mi"
    cpu: "500m"

autoscaling:
  enabled: true
  minReplicas: 2
  maxReplicas: 5
  targetCPUUtilizationPercentage: 70

healthCheck:
  readiness:
    path: /ready
    port: 8001
    initialDelaySeconds: 5
    periodSeconds: 10
  liveness:
    path: /health
    port: 8001
    initialDelaySeconds: 10
    periodSeconds: 15

rollingUpdate:
  maxSurge: 1
  maxUnavailable: 0
```

5. Crear `charts/order-service/templates/deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Chart.Name }}
  namespace: {{ .Release.Namespace }}
  labels:
    app: {{ .Chart.Name }}
    version: {{ .Values.image.tag | quote }}
spec:
  replicas: {{ .Values.replicaCount }}
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: {{ .Values.rollingUpdate.maxSurge }}
      maxUnavailable: {{ .Values.rollingUpdate.maxUnavailable }}
  selector:
    matchLabels:
      app: {{ .Chart.Name }}
  template:
    metadata:
      labels:
        app: {{ .Chart.Name }}
        version: {{ .Values.image.tag | quote }}
    spec:
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - containerPort: {{ .Values.service.port }}
              protocol: TCP
          resources:
            requests:
              memory: {{ .Values.resources.requests.memory }}
              cpu: {{ .Values.resources.requests.cpu }}
            limits:
              memory: {{ .Values.resources.limits.memory }}
              cpu: {{ .Values.resources.limits.cpu }}
          readinessProbe:
            httpGet:
              path: {{ .Values.healthCheck.readiness.path }}
              port: {{ .Values.healthCheck.readiness.port }}
            initialDelaySeconds: {{ .Values.healthCheck.readiness.initialDelaySeconds }}
            periodSeconds: {{ .Values.healthCheck.readiness.periodSeconds }}
          livenessProbe:
            httpGet:
              path: {{ .Values.healthCheck.liveness.path }}
              port: {{ .Values.healthCheck.liveness.port }}
            initialDelaySeconds: {{ .Values.healthCheck.liveness.initialDelaySeconds }}
            periodSeconds: {{ .Values.healthCheck.liveness.periodSeconds }}
```

6. Crear `charts/order-service/templates/service.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ .Chart.Name }}
  namespace: {{ .Release.Namespace }}
  labels:
    app: {{ .Chart.Name }}
spec:
  type: {{ .Values.service.type }}
  ports:
    - port: {{ .Values.service.port }}
      targetPort: {{ .Values.service.port }}
      protocol: TCP
      name: http
  selector:
    app: {{ .Chart.Name }}
```

7. Crear `charts/order-service/templates/hpa.yaml`:

```yaml
{{- if .Values.autoscaling.enabled }}
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: {{ .Chart.Name }}
  namespace: {{ .Release.Namespace }}
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: {{ .Chart.Name }}
  minReplicas: {{ .Values.autoscaling.minReplicas }}
  maxReplicas: {{ .Values.autoscaling.maxReplicas }}
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: {{ .Values.autoscaling.targetCPUUtilizationPercentage }}
{{- end }}
```

8. Validar el chart con Helm:

```bash
helm lint charts/order-service/
helm template order-service charts/order-service/ --namespace microservices-prod
```

9. Commit inicial:

```bash
git add -A
git commit -m "feat: initial Helm chart for order-service GitOps deployment"
```

### Salida Esperada

```
==> Linting charts/order-service/
[INFO] Chart.yaml: icon is recommended

1 chart(s) linted, 0 chart(s) failed
```

### Verificación

```bash
# Verificar que el template renderiza correctamente
helm template order-service charts/order-service/ --namespace microservices-prod | grep "image:"
```

Debe mostrar: `image: "localhost:5050/order-service:latest"`

---

## Paso 4: Crear el Workflow de GitHub Actions

**Objetivo:** Definir el pipeline CI/CD multi-etapa con quality gates, escaneo de seguridad y actualización automática del repositorio GitOps.

### Instrucciones

1. Crear el workflow principal en el repositorio de aplicación:

```bash
cd ~/microservices-lab/lab-11/ms-app-repo
mkdir -p .github/workflows
```

2. Crear `.github/workflows/ci-cd.yml`:

```yaml
name: CI/CD Pipeline - Order Service

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

env:
  REGISTRY: localhost:5050
  IMAGE_NAME: order-service
  GITOPS_REPO_PATH: /home/runner/microservices-lab/lab-11/ms-gitops-repo

jobs:
  # ═══════════════════════════════════════════════════════
  # ETAPA 1: CI - Tests y Calidad de Código
  # ═══════════════════════════════════════════════════════
  ci-test:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout código
        uses: actions/checkout@v4

      - name: Setup Python 3.12
        uses: actions/setup-python@v5
        with:
          python-version: "3.12"

      - name: Instalar dependencias
        run: |
          pip install --upgrade pip
          pip install -r requirements.txt
          pip install -r requirements-dev.txt

      - name: Ejecutar tests con cobertura
        run: |
          pytest tests/ \
            --cov=src \
            --cov-report=term-missing \
            --cov-report=xml:coverage.xml \
            --cov-fail-under=80 \
            --junitxml=test-results.xml \
            -v

      - name: Publicar resultados de tests
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name: test-results
          path: |
            test-results.xml
            coverage.xml

  # ═══════════════════════════════════════════════════════
  # ETAPA 2: Build de Imagen Docker
  # ═══════════════════════════════════════════════════════
  build:
    runs-on: ubuntu-latest
    needs: ci-test
    outputs:
      image_tag: ${{ steps.meta.outputs.sha_tag }}
    steps:
      - name: Checkout código
        uses: actions/checkout@v4

      - name: Generar metadata de imagen
        id: meta
        run: |
          SHA_TAG=$(git rev-parse --short HEAD)
          echo "sha_tag=${SHA_TAG}" >> $GITHUB_OUTPUT
          echo "Tag de imagen: ${SHA_TAG}"

      - name: Build imagen Docker
        run: |
          docker build \
            -t ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ steps.meta.outputs.sha_tag }} \
            -t ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:latest \
            .

      - name: Push imagen al registro local
        run: |
          docker push ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ steps.meta.outputs.sha_tag }}
          docker push ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:latest

  # ═══════════════════════════════════════════════════════
  # ETAPA 3: Escaneo de Seguridad
  # ═══════════════════════════════════════════════════════
  security-scan:
    runs-on: ubuntu-latest
    needs: build
    steps:
      - name: Escaneo Trivy - Vulnerabilidades
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: "${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ needs.build.outputs.image_tag }}"
          format: "table"
          exit-code: "1"
          severity: "CRITICAL"
          ignore-unfixed: true

      - name: Escaneo Trivy - Reporte SARIF
        uses: aquasecurity/trivy-action@master
        if: always()
        with:
          image-ref: "${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ needs.build.outputs.image_tag }}"
          format: "sarif"
          output: "trivy-results.sarif"
          severity: "CRITICAL,HIGH"

  # ═══════════════════════════════════════════════════════
  # ETAPA 4: Actualización GitOps
  # ═══════════════════════════════════════════════════════
  gitops-update:
    runs-on: ubuntu-latest
    needs: [build, security-scan]
    if: github.ref == 'refs/heads/main'
    steps:
      - name: Actualizar image tag en GitOps repo
        run: |
          cd ${{ env.GITOPS_REPO_PATH }}
          IMAGE_TAG=${{ needs.build.outputs.image_tag }}
          
          # Actualizar el tag en values.yaml
          sed -i "s/^  tag: .*/  tag: \"${IMAGE_TAG}\"/" charts/order-service/values.yaml
          
          # Commit y push del cambio
          git config user.email "ci-bot@mslab.local"
          git config user.name "CI Bot"
          git add charts/order-service/values.yaml
          git commit -m "chore: update order-service image to ${IMAGE_TAG}

          Triggered by app-repo commit: ${{ github.sha }}
          Pipeline run: ${{ github.run_id }}"
```

3. Crear un script de ejecución local que simula el pipeline (para uso con `act` o ejecución directa):

```bash
cat > scripts/run-pipeline-local.sh << 'EOF'
#!/bin/bash
set -euo pipefail

# ══════════════════════════════════════════════════════════════
# Script de Pipeline CI/CD Local
# Simula el workflow de GitHub Actions para ejecución local
# ══════════════════════════════════════════════════════════════

SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
APP_REPO_DIR="$(dirname "$SCRIPT_DIR")"
GITOPS_REPO_DIR="$HOME/microservices-lab/lab-11/ms-gitops-repo"
REGISTRY="localhost:5050"
IMAGE_NAME="order-service"

echo "═══════════════════════════════════════════════════"
echo "  ETAPA 1: CI - Tests y Cobertura"
echo "═══════════════════════════════════════════════════"

cd "$APP_REPO_DIR"
pip install -r requirements.txt -r requirements-dev.txt -q

pytest tests/ \
  --cov=src \
  --cov-report=term-missing \
  --cov-fail-under=80 \
  --junitxml=test-results.xml \
  -v

if [ $? -ne 0 ]; then
  echo "✗ QUALITY GATE FALLIDO: Cobertura < 80% o tests fallidos"
  exit 1
fi
echo "✓ Tests pasados con cobertura ≥ 80%"

echo ""
echo "═══════════════════════════════════════════════════"
echo "  ETAPA 2: Build de Imagen Docker"
echo "═══════════════════════════════════════════════════"

IMAGE_TAG=$(git rev-parse --short HEAD)
echo "Tag de imagen: ${IMAGE_TAG}"

docker build \
  -t ${REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG} \
  -t ${REGISTRY}/${IMAGE_NAME}:latest \
  .

echo "✓ Imagen construida: ${REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG}"

echo ""
echo "═══════════════════════════════════════════════════"
echo "  ETAPA 3: Escaneo de Seguridad con Trivy"
echo "═══════════════════════════════════════════════════"

# Instalar Trivy si no está disponible
if ! command -v trivy &> /dev/null; then
  echo "Instalando Trivy..."
  curl -sfL https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/install.sh | sh -s -- -b /usr/local/bin v0.52.2
fi

trivy image --exit-code 1 --severity CRITICAL --ignore-unfixed \
  ${REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG}

TRIVY_EXIT=$?
if [ $TRIVY_EXIT -ne 0 ]; then
  echo "✗ QUALITY GATE FALLIDO: Vulnerabilidades CRITICAL detectadas"
  exit 1
fi
echo "✓ Sin vulnerabilidades CRITICAL"

echo ""
echo "═══════════════════════════════════════════════════"
echo "  ETAPA 4: Push al Registro"
echo "═══════════════════════════════════════════════════"

docker push ${REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG}
docker push ${REGISTRY}/${IMAGE_NAME}:latest
echo "✓ Imagen publicada en ${REGISTRY}"

echo ""
echo "═══════════════════════════════════════════════════"
echo "  ETAPA 5: Actualización GitOps"
echo "═══════════════════════════════════════════════════"

cd "$GITOPS_REPO_DIR"
sed -i "s/^  tag: .*/  tag: \"${IMAGE_TAG}\"/" charts/order-service/values.yaml
git add charts/order-service/values.yaml
git commit -m "chore: update order-service image to ${IMAGE_TAG}" || echo "Sin cambios"

echo "✓ GitOps repo actualizado con tag: ${IMAGE_TAG}"
echo ""
echo "═══════════════════════════════════════════════════"
echo "  ✓ PIPELINE COMPLETADO EXITOSAMENTE"
echo "═══════════════════════════════════════════════════"
echo "Imagen: ${REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG}"
echo "GitOps values.yaml actualizado"
EOF

chmod +x scripts/run-pipeline-local.sh
```

4. Crear directorio de scripts y commit:

```bash
mkdir -p scripts
# El script ya fue creado arriba
git add -A
git commit -m "feat: add CI/CD workflow and local pipeline script"
```

### Salida Esperada

Al ejecutar `cat .github/workflows/ci-cd.yml | head -5`:
```
name: CI/CD Pipeline - Order Service

on:
  push:
    branches: [main]
```

### Verificación

```bash
# Validar sintaxis YAML del workflow
python -c "import yaml; yaml.safe_load(open('.github/workflows/ci-cd.yml'))" && echo "✓ YAML válido"
```

---

## Paso 5: Instalar y Configurar Argo CD

**Objetivo:** Desplegar Argo CD 2.11.3 en el clúster Kubernetes como operador GitOps que monitorea el repositorio de configuración y reconcilia el estado del clúster automáticamente.

### Instrucciones

1. Crear el namespace y desplegar Argo CD:

```bash
# Crear namespace para Argo CD
kubectl create namespace argocd

# Instalar Argo CD via Helm
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update

helm install argocd argo/argo-cd \
  --namespace argocd \
  --version 7.3.3 \
  --set server.service.type=NodePort \
  --set server.service.nodePortHttp=30080 \
  --set configs.params."server\.insecure"=true \
  --wait --timeout 300s
```

2. Esperar a que todos los pods estén listos:

```bash
kubectl wait --for=condition=Ready pods --all -n argocd --timeout=300s
kubectl get pods -n argocd
```

3. Obtener la contraseña inicial de admin:

```bash
ARGOCD_PASSWORD=$(kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d)
echo "Argo CD Admin Password: ${ARGOCD_PASSWORD}"
```

4. Instalar el CLI de Argo CD:

```bash
# Descargar CLI de Argo CD
curl -sSL -o /usr/local/bin/argocd \
  https://github.com/argoproj/argo-cd/releases/download/v2.11.3/argocd-linux-amd64
chmod +x /usr/local/bin/argocd

# Verificar instalación
argocd version --client
```

5. Configurar acceso al servidor Argo CD:

```bash
# Obtener URL de acceso
ARGOCD_URL=$(minikube service argocd-server -n argocd --url | head -1)
echo "Argo CD URL: ${ARGOCD_URL}"

# Login con CLI (usando port-forward como alternativa más fiable)
kubectl port-forward svc/argocd-server -n argocd 8080:443 &
ARGOCD_PF_PID=$!
sleep 3

argocd login localhost:8080 \
  --username admin \
  --password "${ARGOCD_PASSWORD}" \
  --insecure

# Detener port-forward temporal
kill $ARGOCD_PF_PID 2>/dev/null || true
```

6. Crear el namespace de producción:

```bash
kubectl create namespace microservices-prod
kubectl label namespace microservices-prod env=production
```

### Salida Esperada

```
NAME                                               READY   STATUS    RESTARTS   AGE
argocd-application-controller-0                    1/1     Running   0          60s
argocd-applicationset-controller-xxxxxxxxx-xxxxx   1/1     Running   0          60s
argocd-dex-server-xxxxxxxxx-xxxxx                  1/1     Running   0          60s
argocd-notifications-controller-xxxxxxxxx-xxxxx    1/1     Running   0          60s
argocd-redis-xxxxxxxxx-xxxxx                       1/1     Running   0          60s
argocd-repo-server-xxxxxxxxx-xxxxx                 1/1     Running   0          60s
argocd-server-xxxxxxxxx-xxxxx                      1/1     Running   0          60s
```

### Verificación

```bash
argocd version --client
kubectl get svc -n argocd argocd-server
```

---

## Paso 6: Configurar Argo CD Application con GitOps

**Objetivo:** Crear el recurso Application CRD de Argo CD que conecta el repositorio GitOps con el namespace de producción, habilitando sincronización automática con self-heal y prune.

### Instrucciones

1. Dado que usamos repositorios Git locales, necesitamos exponerlos a Argo CD. Configurar un servidor Git local dentro del clúster:

```bash
cd ~/microservices-lab/lab-11

# Crear un bare repo para GitOps que Argo CD pueda acceder
mkdir -p /tmp/git-server/ms-gitops-repo.git
cd /tmp/git-server/ms-gitops-repo.git
git init --bare

# Push del contenido del gitops repo al bare repo
cd ~/microservices-lab/lab-11/ms-gitops-repo
git remote add local-bare /tmp/git-server/ms-gitops-repo.git
git push local-bare main
```

2. Desplegar un servidor Git ligero en el clúster:

```bash
cat << 'EOF' > /tmp/git-server-deployment.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: gitops-repo-data
  namespace: argocd
data: {}
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: git-server
  namespace: argocd
spec:
  replicas: 1
  selector:
    matchLabels:
      app: git-server
  template:
    metadata:
      labels:
        app: git-server
    spec:
      containers:
        - name: git-daemon
          image: alpine/git:latest
          command: ["git", "daemon", "--reuseaddr", "--base-path=/git", "--export-all", "--enable=receive-pack", "/git"]
          ports:
            - containerPort: 9418
          volumeMounts:
            - name: git-repos
              mountPath: /git
      initContainers:
        - name: init-repo
          image: alpine/git:latest
          command:
            - sh
            - -c
            - |
              cd /git
              git init --bare ms-gitops-repo.git
              cd ms-gitops-repo.git
              git config receive.denyCurrentBranch ignore
          volumeMounts:
            - name: git-repos
              mountPath: /git
      volumes:
        - name: git-repos
          emptyDir: {}
---
apiVersion: v1
kind: Service
metadata:
  name: git-server
  namespace: argocd
spec:
  selector:
    app: git-server
  ports:
    - port: 9418
      targetPort: 9418
EOF

kubectl apply -f /tmp/git-server-deployment.yaml
kubectl wait --for=condition=Ready pod -l app=git-server -n argocd --timeout=120s
```

3. Inicializar el repositorio en el servidor Git del clúster empujando nuestro chart:

```bash
# Usar port-forward para acceder al git-server
kubectl port-forward svc/git-server -n argocd 9418:9418 &
GIT_PF_PID=$!
sleep 2

cd ~/microservices-lab/lab-11/ms-gitops-repo
git remote add cluster git://localhost:9418/ms-gitops-repo.git || \
  git remote set-url cluster git://localhost:9418/ms-gitops-repo.git
git push cluster main --force

kill $GIT_PF_PID 2>/dev/null || true
```

4. Crear el recurso Application de Argo CD:

```bash
cat << 'EOF' > /tmp/argocd-application.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: order-service
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: git://git-server.argocd.svc.cluster.local:9418/ms-gitops-repo.git
    targetRevision: main
    path: charts/order-service
    helm:
      valueFiles:
        - values.yaml
  destination:
    server: https://kubernetes.default.svc
    namespace: microservices-prod
  syncPolicy:
    automated:
      selfHeal: true
      prune: true
    syncOptions:
      - CreateNamespace=true
      - ApplyOutOfSyncOnly=true
    retry:
      limit: 3
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 1m
  # Health checks personalizados
  ignoreDifferences:
    - group: apps
      kind: Deployment
      jsonPointers:
        - /spec/replicas  # Ignorar diferencias por HPA
EOF

kubectl apply -f /tmp/argocd-application.yaml
```

5. Verificar que Argo CD detecta la aplicación:

```bash
# Port-forward para CLI
kubectl port-forward svc/argocd-server -n argocd 8080:443 &
ARGOCD_PF_PID=$!
sleep 3

argocd app get order-service --insecure

kill $ARGOCD_PF_PID 2>/dev/null || true
```

### Salida Esperada

```
Name:               argocd/order-service
Project:            default
Server:             https://kubernetes.default.svc
Namespace:          microservices-prod
URL:                https://localhost:8080/applications/order-service
Repo:               git://git-server.argocd.svc.cluster.local:9418/ms-gitops-repo.git
Target:             main
Path:               charts/order-service
SyncWindow:         Sync Allowed
Sync Policy:        Automated (Prune)
Sync Status:        Synced
Health Status:      Progressing
```

### Verificación

```bash
# Verificar que la aplicación fue creada
kubectl get applications -n argocd
# Verificar que los recursos se están creando en microservices-prod
kubectl get all -n microservices-prod
```

---

## Paso 7: Ejecutar el Pipeline Completo End-to-End

**Objetivo:** Construir la imagen inicial, publicarla al registro y verificar que Argo CD despliega automáticamente el servicio en Kubernetes con health checks operativos.

### Instrucciones

1. Ejecutar la primera construcción y publicación de la imagen:

```bash
cd ~/microservices-lab/lab-11/ms-app-repo

# Obtener el SHA del commit actual
IMAGE_TAG=$(git rev-parse --short HEAD)
echo "Construyendo imagen con tag: ${IMAGE_TAG}"

# Build de la imagen
docker build \
  -t localhost:5050/order-service:${IMAGE_TAG} \
  -t localhost:5050/order-service:latest \
  .

# Push al registro local
docker push localhost:5050/order-service:${IMAGE_TAG}
docker push localhost:5050/order-service:latest
```

2. Actualizar el repositorio GitOps con el tag real:

```bash
cd ~/microservices-lab/lab-11/ms-gitops-repo

# Actualizar el tag en values.yaml
sed -i "s/^  tag: .*/  tag: \"${IMAGE_TAG}\"/" charts/order-service/values.yaml

# Verificar el cambio
grep "tag:" charts/order-service/values.yaml

# Commit y push
git add charts/order-service/values.yaml
git commit -m "chore: deploy order-service with tag ${IMAGE_TAG}"

# Push al servidor Git del clúster
kubectl port-forward svc/git-server -n argocd 9418:9418 &
GIT_PF_PID=$!
sleep 2
git push cluster main
kill $GIT_PF_PID 2>/dev/null || true
```

3. Forzar sincronización de Argo CD (o esperar al intervalo de 3 minutos):

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443 &
ARGOCD_PF_PID=$!
sleep 3

argocd app sync order-service --insecure
argocd app wait order-service --health --timeout 120 --insecure

kill $ARGOCD_PF_PID 2>/dev/null || true
```

4. Verificar el despliegue en Kubernetes:

```bash
# Verificar pods
kubectl get pods -n microservices-prod -l app=order-service

# Verificar el rollout
kubectl rollout status deployment/order-service -n microservices-prod --timeout=120s

# Verificar que la imagen correcta está desplegada
kubectl get deployment order-service -n microservices-prod \
  -o jsonpath='{.spec.template.spec.containers[0].image}'
echo ""
```

5. Probar el servicio desplegado:

```bash
# Port-forward al servicio
kubectl port-forward svc/order-service -n microservices-prod 8001:8001 &
SVC_PF_PID=$!
sleep 3

# Test de health
curl -s http://localhost:8001/health | python -m json.tool

# Test de readiness
curl -s http://localhost:8001/ready | python -m json.tool

# Crear un pedido
curl -s -X POST http://localhost:8001/api/v1/orders \
  -H "Content-Type: application/json" \
  -d '{"product_id":"PROD-001","quantity":5,"customer_id":"CUST-100"}' | python -m json.tool

kill $SVC_PF_PID 2>/dev/null || true
```

### Salida Esperada

```json
{
    "status": "healthy",
    "timestamp": "2024-XX-XXTXX:XX:XX.XXXXXX"
}
```

```json
{
    "order_id": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
    "product_id": "PROD-001",
    "quantity": 5,
    "customer_id": "CUST-100",
    "status": "pending",
    "created_at": "2024-XX-XXTXX:XX:XX.XXXXXX"
}
```

### Verificación

```bash
# Verificar que todos los pods están Running y Ready
kubectl get pods -n microservices-prod -l app=order-service \
  -o custom-columns="NAME:.metadata.name,STATUS:.status.phase,READY:.status.conditions[?(@.type=='Ready')].status"
```

Todos los pods deben mostrar `STATUS: Running` y `READY: True`.

---

## Paso 8: Demostrar Ciclo Completo de Cambio con Pipeline Local

**Objetivo:** Realizar un cambio en el código del microservicio, ejecutar el pipeline local completo y verificar que Argo CD despliega automáticamente la nueva versión con rolling update.

### Instrucciones

1. Realizar un cambio en el código del order-service:

```bash
cd ~/microservices-lab/lab-11/ms-app-repo

# Agregar un nuevo endpoint al servicio
cat >> src/order_service/main.py << 'EOF'

@app.get("/api/v1/orders/stats/summary")
def get_orders_summary():
    """Endpoint añadido para demostrar pipeline CI/CD end-to-end"""
    total_orders = len(orders_db)
    pending = sum(1 for o in orders_db.values() if o.status == "pending")
    return {
        "total_orders": total_orders,
        "pending_orders": pending,
        "service_version": "1.1.0"
    }
EOF
```

2. Agregar test para el nuevo endpoint:

```bash
cat >> tests/test_orders.py << 'EOF'

class TestOrderStats:
    def test_orders_summary_empty(self, client):
        response = client.get("/api/v1/orders/stats/summary")
        assert response.status_code == 200
        data = response.json()
        assert data["total_orders"] == 0
        assert data["service_version"] == "1.1.0"

    def test_orders_summary_with_data(self, client):
        # Crear pedidos
        for i in range(3):
            client.post("/api/v1/orders", json={
                "product_id": f"PROD-{i}",
                "quantity": 1,
                "customer_id": "CUST-400"
            })
        response = client.get("/api/v1/orders/stats/summary")
        data = response.json()
        assert data["total_orders"] == 3
        assert data["pending_orders"] == 3
EOF
```

3. Commit del cambio:

```bash
git add -A
git commit -m "feat: add orders summary endpoint v1.1.0"
```

4. Ejecutar el pipeline local:

```bash
./scripts/run-pipeline-local.sh
```

5. Sincronizar con el servidor Git del clúster y esperar despliegue:

```bash
cd ~/microservices-lab/lab-11/ms-gitops-repo

# Push al servidor Git del clúster
kubectl port-forward svc/git-server -n argocd 9418:9418 &
GIT_PF_PID=$!
sleep 2
git push cluster main
kill $GIT_PF_PID 2>/dev/null || true

# Sincronizar Argo CD
kubectl port-forward svc/argocd-server -n argocd 8080:443 &
ARGOCD_PF_PID=$!
sleep 3

argocd app sync order-service --insecure
argocd app wait order-service --health --timeout 120 --insecure

kill $ARGOCD_PF_PID 2>/dev/null || true
```

6. Verificar el rolling update:

```bash
# Observar el rollout
kubectl rollout status deployment/order-service -n microservices-prod

# Verificar la nueva imagen
kubectl get deployment order-service -n microservices-prod \
  -o jsonpath='{.spec.template.spec.containers[0].image}'
echo ""

# Verificar que el nuevo endpoint está disponible
kubectl port-forward svc/order-service -n microservices-prod 8001:8001 &
SVC_PF_PID=$!
sleep 3

curl -s http://localhost:8001/api/v1/orders/stats/summary | python -m json.tool

kill $SVC_PF_PID 2>/dev/null || true
```

### Salida Esperada

Pipeline local:
```
═══════════════════════════════════════════════════
  ETAPA 1: CI - Tests y Cobertura
═══════════════════════════════════════════════════
...
8 passed
✓ Tests pasados con cobertura ≥ 80%

═══════════════════════════════════════════════════
  ETAPA 2: Build de Imagen Docker
═══════════════════════════════════════════════════
Tag de imagen: a1b2c3d
✓ Imagen construida: localhost:5050/order-service:a1b2c3d

═══════════════════════════════════════════════════
  ETAPA 3: Escaneo de Seguridad con Trivy
═══════════════════════════════════════════════════
✓ Sin vulnerabilidades CRITICAL

═══════════════════════════════════════════════════
  ETAPA 4: Push al Registro
═══════════════════════════════════════════════════
✓ Imagen publicada en localhost:5050

═══════════════════════════════════════════════════
  ETAPA 5: Actualización GitOps
═══════════════════════════════════════════════════
✓ GitOps repo actualizado con tag: a1b2c3d

═══════════════════════════════════════════════════
  ✓ PIPELINE COMPLETADO EXITOSAMENTE
═══════════════════════════════════════════════════
```

Nuevo endpoint:
```json
{
    "total_orders": 0,
    "pending_orders": 0,
    "service_version": "1.1.0"
}
```

### Verificación

```bash
# Verificar historial de revisiones del deployment
kubectl rollout history deployment/order-service -n microservices-prod

# Verificar que Argo CD muestra Synced y Healthy
kubectl port-forward svc/argocd-server -n argocd 8080:443 &
ARGOCD_PF_PID=$!
sleep 3
argocd app get order-service --insecure | grep -E "Sync Status|Health Status"
kill $ARGOCD_PF_PID 2>/dev/null || true
```

Debe mostrar:
```
Sync Status:        Synced
Health Status:      Healthy
```

---

## Paso 9: Verificar Self-Healing y Rollback Automático

**Objetivo:** Demostrar que Argo CD reconcilia automáticamente el estado del clúster cuando se produce una modificación fuera de banda (drift), y que Kubernetes realiza rollback automático ante fallos de health checks.

### Instrucciones

1. Demostrar self-healing (corrección de drift):

```bash
# Modificar manualmente el número de réplicas (simulando drift)
kubectl scale deployment order-service -n microservices-prod --replicas=5
echo "Réplicas escaladas manualmente a 5"

# Esperar unos segundos para que Argo CD detecte el drift
sleep 30

# Verificar que Argo CD revirtió al estado deseado (2 réplicas)
kubectl get deployment order-service -n microservices-prod \
  -o jsonpath='{.spec.replicas}'
echo " réplicas (debe ser 2)"
```

2. Demostrar rollback automático ante imagen inválida:

```bash
cd ~/microservices-lab/lab-11/ms-gitops-repo

# Simular un despliegue con imagen inexistente
sed -i 's/^  tag: .*/  tag: "nonexistent-broken-tag"/' charts/order-service/values.yaml
git add charts/order-service/values.yaml
git commit -m "test: deploy broken image to verify rollback"

# Push al servidor Git del clúster
kubectl port-forward svc/git-server -n argocd 9418:9418 &
GIT_PF_PID=$!
sleep 2
git push cluster main
kill $GIT_PF_PID 2>/dev/null || true

# Sincronizar
kubectl port-forward svc/argocd-server -n argocd 8080:443 &
ARGOCD_PF_PID=$!
sleep 3
argocd app sync order-service --insecure
sleep 20

# Observar que el deployment no progresa (pods en ImagePullBackOff)
kubectl get pods -n microservices-prod -l app=order-service
echo ""
echo "Los pods antiguos siguen corriendo gracias a maxUnavailable: 0"
```

3. Restaurar la versión correcta (simular corrección):

```bash
# Revertir al tag correcto
IMAGE_TAG=$(cd ~/microservices-lab/lab-11/ms-app-repo && git rev-parse --short HEAD)
cd ~/microservices-lab/lab-11/ms-gitops-repo
sed -i "s/^  tag: .*/  tag: \"${IMAGE_TAG}\"/" charts/order-service/values.yaml
git add charts/order-service/values.yaml
git commit -m "fix: revert to working image tag ${IMAGE_TAG}"

kubectl port-forward svc/git-server -n argocd 9418:9418 &
GIT_PF_PID=$!
sleep 2
git push cluster main
kill $GIT_PF_PID 2>/dev/null || true

# Sincronizar y esperar
argocd app sync order-service --insecure
argocd app wait order-service --health --timeout 120 --insecure

kill $ARGOCD_PF_PID 2>/dev/null || true
```

### Salida Esperada

Self-healing:
```
2 réplicas (debe ser 2)
```

Rollback (pods con imagen rota):
```
NAME                             READY   STATUS             RESTARTS   AGE
order-service-xxxxxxxxx-xxxxx    1/1     Running            0          5m    ← pod antiguo sigue vivo
order-service-yyyyyyyyy-zzzzz    0/1     ImagePullBackOff   0          20s   ← pod nuevo falla
```

### Verificación

```bash
# Estado final: todo saludable
kubectl get pods -n microservices-prod -l app=order-service
kubectl rollout status deployment/order-service -n microservices-prod --timeout=60s
```

---

## Validación y Testing Final

Ejecutar esta secuencia completa de validación para confirmar que todo el sistema funciona correctamente:

```bash
echo "═══════════════════════════════════════════════════"
echo "  VALIDACIÓN FINAL DEL LABORATORIO"
echo "═══════════════════════════════════════════════════"

echo ""
echo "1. Registro Docker local:"
curl -s http://localhost:5050/v2/order-service/tags/list | python -m json.tool

echo ""
echo "2. Pods en microservices-prod:"
kubectl get pods -n microservices-prod -l app=order-service -o wide

echo ""
echo "3. Estado de Argo CD Application:"
kubectl get applications -n argocd order-service -o jsonpath='{.status.sync.status}' && echo " (Sync)"
kubectl get applications -n argocd order-service -o jsonpath='{.status.health.status}' && echo " (Health)"

echo ""
echo "4. Imagen desplegada:"
kubectl get deployment order-service -n microservices-prod \
  -o jsonpath='{.spec.template.spec.containers[0].image}'
echo ""

echo ""
echo "5. Health check del servicio:"
kubectl port-forward svc/order-service -n microservices-prod 8001:8001 &
SVC_PF=$!
sleep 3
curl -s http://localhost:8001/health
echo ""
curl -s http://localhost:8001/api/v1/orders/stats/summary
echo ""
kill $SVC_PF 2>/dev/null || true

echo ""
echo "6. Historial de Git en gitops-repo:"
cd ~/microservices-lab/lab-11/ms-gitops-repo
git log --oneline -5

echo ""
echo "7. Rolling update strategy:"
kubectl get deployment order-service -n microservices-prod \
  -o jsonpath='{.spec.strategy}' | python -m json.tool

echo ""
echo "═══════════════════════════════════════════════════"
echo "  ✓ VALIDACIÓN COMPLETADA"
echo "═══════════════════════════════════════════════════"
```

### Criterios de Éxito

| Criterio | Condición |
|----------|-----------|
| Registro Docker | Contiene al menos 2 tags de order-service |
| Pods | 2 réplicas Running y Ready |
| Argo CD Sync | Estado `Synced` |
| Argo CD Health | Estado `Healthy` |
| Health endpoint | Responde `{"status": "healthy"}` |
| Pipeline tests | Cobertura ≥ 80% |
| Self-healing | Réplicas vuelven a 2 tras drift |
| Rolling update | `maxUnavailable: 0` configurado |

---

## Resolución de Problemas

### Problema 1: Argo CD no puede clonar el repositorio Git

**Síntomas:**
- La aplicación en Argo CD muestra estado `Unknown` o `ComparisonError`
- En los logs del repo-server: `rpc error: code = Unknown desc = error creating SSH agent`
- `argocd app get order-service` muestra: `Failed to load target state: failed to generate manifest`

**Causa:** El servidor Git interno no es accesible desde el pod `argocd-repo-server`, o el protocolo git:// está bloqueado por una NetworkPolicy.

**Solución:**

```bash
# 1. Verificar que el git-server está corriendo
kubectl get pods -n argocd -l app=git-server
kubectl logs -n argocd -l app=git-server

# 2. Verificar conectividad desde argocd-repo-server
kubectl exec -n argocd deploy/argocd-repo-server -- \
  git ls-remote git://git-server.argocd.svc.cluster.local:9418/ms-gitops-repo.git

# 3. Si falla, verificar el servicio DNS
kubectl exec -n argocd deploy/argocd-repo-server -- \
  nslookup git-server.argocd.svc.cluster.local

# 4. Alternativa: usar ConfigMap con manifiestos directos
# Registrar el repo manualmente en Argo CD
kubectl port-forward svc/argocd-server -n argocd 8080:443 &
sleep 3
argocd repo add git://git-server.argocd.svc.cluster.local:9418/ms-gitops-repo.git \
  --insecure-skip-server-verification --insecure
```

### Problema 2: Pipeline falla en escaneo Trivy con vulnerabilidades en imagen base

**Síntomas:**
- El script `run-pipeline-local.sh` falla en la Etapa 3
- Trivy reporta vulnerabilidades CRITICAL en paquetes del sistema operativo de `python:3.12.4-slim`
- Mensaje: `QUALITY GATE FALLIDO: Vulnerabilidades CRITICAL detectadas`

**Causa:** La imagen base `python:3.12.4-slim` contiene paquetes de sistema con CVEs conocidos que aún no tienen parche disponible upstream.

**Solución:**

```bash
# 1. Identificar las vulnerabilidades específicas
trivy image --severity CRITICAL localhost:5050/order-service:latest

# 2. Opción A: Ignorar vulnerabilidades sin fix disponible (recomendado para labs)
# Modificar el script para usar --ignore-unfixed
# En scripts/run-pipeline-local.sh, la línea de trivy ya incluye --ignore-unfixed

# 3. Opción B: Crear un archivo .trivyignore para CVEs específicos sin parche
cat > ~/microservices-lab/lab-11/ms-app-repo/.trivyignore << 'EOF'
# CVEs sin parche disponible en python:3.12.4-slim - revisado 2024-06
CVE-2024-XXXXX
CVE-2024-YYYYY
EOF

# 4. Opción C: Usar una imagen base más reciente
# En el Dockerfile, cambiar a:
# FROM python:3.12.5-slim AS builder
# (si está disponible con parches)

# 5. Re-ejecutar el pipeline
cd ~/microservices-lab/lab-11/ms-app-repo
./scripts/run-pipeline-local.sh
```

---

## Limpieza

```bash
# Eliminar la aplicación de Argo CD
kubectl port-forward svc/argocd-server -n argocd 8080:443 &
sleep 3
argocd app delete order-service --cascade --insecure -y
kill %1 2>/dev/null || true

# Eliminar recursos de Kubernetes
kubectl delete namespace microservices-prod
helm uninstall argocd -n argocd
kubectl delete namespace argocd

# Eliminar el git-server
kubectl delete -f /tmp/git-server-deployment.yaml 2>/dev/null || true

# Detener y eliminar el registro local
docker stop local-registry && docker rm local-registry

# Limpiar imágenes Docker locales
docker rmi $(docker images "localhost:5050/order-service" -q) 2>/dev/null || true
docker rmi $(docker images "localhost:5050/test-image" -q) 2>/dev/null || true

# Eliminar repositorios del laboratorio (opcional - mantener si se continúa al lab 12)
# rm -rf ~/microservices-lab/lab-11/ms-app-repo
# rm -rf ~/microservices-lab/lab-11/ms-gitops-repo
# rm -rf /tmp/git-server

echo "✓ Limpieza completada"
```

---

## Resumen

En este laboratorio implementaste un pipeline CI/CD completo siguiendo principios GitOps:

| Componente | Implementación |
|------------|----------------|
| **Repositorio de aplicación** | `ms-app-repo` con código, tests, Dockerfile y workflow |
| **Repositorio GitOps** | `ms-gitops-repo` con Helm chart declarativo |
| **Pipeline CI** | 5 etapas: test → build → scan → push → gitops-update |
| **Quality Gates** | Cobertura ≥80%, cero CRITICAL en Trivy |
| **Operador GitOps** | Argo CD con syncPolicy automated, selfHeal, prune |
| **Despliegue progresivo** | Rolling update con maxUnavailable: 0 |
| **Self-healing** | Reconciliación automática ante drift |

**Principios aplicados:**

1. **Separación de concerns**: Código de aplicación y configuración de infraestructura en repositorios independientes
2. **Infraestructura declarativa**: Todo el estado deseado expresado en YAML versionado
3. **Artefacto inmutable**: La misma imagen Docker fluye desde CI hasta producción
4. **Reconciliación continua**: Argo CD mantiene el clúster alineado con Git como fuente de verdad
5. **Gates de calidad**: Ningún artefacto defectuoso llega a producción

### Recursos Adicionales

- [Especificación OpenGitOps](https://opengitops.dev/)
- [Argo CD Documentation](https://argo-cd.readthedocs.io/en/stable/)
- [GitHub Actions Workflow Syntax](https://docs.github.com/en/actions/using-workflows/workflow-syntax-for-github-actions)
- [Trivy Scanner Documentation](https://aquasecurity.github.io/trivy/)
- [Helm Best Practices](https://helm.sh/docs/chart_best_practices/)
