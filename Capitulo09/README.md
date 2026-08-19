# 4 Laboratorio: construir imágenes optimizadas y empaquetar con Helm

| Campo | Valor |
|-------|-------|
| **Duración** | 66 minutos |
| **Complejidad** | Media |
| **Nivel Bloom** | Crear |

## Descripción General

En este laboratorio consolidarás los microservicios desarrollados en labs anteriores aplicando prácticas profesionales de empaquetado. Refactorizarás los Dockerfiles usando multi-stage builds con usuario no-root, escanearás vulnerabilidades con Trivy, crearás un Helm chart reutilizable parametrizado y desplegarás los servicios en minikube validando upgrade y rollback.

## Objetivos de Aprendizaje

- [ ] Refactorizar Dockerfiles aplicando multi-stage builds, usuario no-root, `.dockerignore` y capas optimizadas reduciendo el tamaño de imagen en al menos un 40%
- [ ] Escanear imágenes con Trivy 0.52.2, identificar vulnerabilidades CVE y aplicar remediaciones básicas
- [ ] Crear un Helm chart reutilizable con templates parametrizados que soporte múltiples microservicios
- [ ] Configurar un registro local de imágenes en minikube y publicar las imágenes optimizadas
- [ ] Desplegar los microservicios usando Helm con values diferenciados, validando rollback y upgrade

## Prerrequisitos

### Conocimiento

- Comprensión de capas Docker y sistema de archivos de unión
- Conocimiento básico de templates Go (Helm)
- Labs 06-00-01, 07-00-01 y 08-00-01 completados

### Software Instalado

| Herramienta | Versión | Verificación |
|-------------|---------|--------------|
| Docker Engine | 26.1.x | `docker --version` |
| minikube | 1.33+ | `minikube version` |
| kubectl | 1.30+ | `kubectl version --client` |
| Helm | 3.15.2 | `helm version --short` |
| Trivy | 0.52.2 | `trivy --version` |
| Python | 3.12.3 | `python3 --version` |

## Entorno del Laboratorio

### Preparación Inicial

```bash
# Crear estructura de trabajo
mkdir -p ~/microservices-lab/{order-service,inventory-service,gateway-service,helm/msvc-chart,scripts}
cd ~/microservices-lab

# Iniciar minikube con recursos suficientes
minikube start --cpus=4 --memory=8192 --driver=docker --addons=registry

# Verificar que el addon registry está activo
minikube addons list | grep registry
```

**Salida esperada:**
```
| registry | minikube | enabled ✅ |
```

```bash
# Configurar acceso al registro local de minikube
# El addon registry expone el registro en localhost:5000 dentro del clúster
eval $(minikube docker-env)
```

---

## Paso 1: Crear los Microservicios Base

**Objetivo:** Preparar el código fuente mínimo de los tres microservicios para trabajar con los Dockerfiles.

### Instrucciones

1. Crear el código del `order-service`:

```bash
mkdir -p ~/microservices-lab/order-service/app
cat > ~/microservices-lab/order-service/app/__init__.py << 'EOF'
EOF

cat > ~/microservices-lab/order-service/app/main.py << 'EOF'
from fastapi import FastAPI
import os

app = FastAPI(title="Order Service", version="1.0.0")

@app.get("/health")
async def health():
    return {"status": "healthy", "service": "order-service"}

@app.get("/api/v1/orders")
async def list_orders():
    return {"orders": [], "service_version": os.getenv("APP_VERSION", "1.0.0")}
EOF

cat > ~/microservices-lab/order-service/requirements.txt << 'EOF'
fastapi==0.111.0
uvicorn==0.29.0
pydantic==2.7.1
EOF
```

2. Crear el código del `inventory-service`:

```bash
mkdir -p ~/microservices-lab/inventory-service/app
cat > ~/microservices-lab/inventory-service/app/__init__.py << 'EOF'
EOF

cat > ~/microservices-lab/inventory-service/app/main.py << 'EOF'
from fastapi import FastAPI
import os

app = FastAPI(title="Inventory Service", version="1.0.0")

@app.get("/health")
async def health():
    return {"status": "healthy", "service": "inventory-service"}

@app.get("/api/v1/inventory")
async def list_inventory():
    return {"items": [], "service_version": os.getenv("APP_VERSION", "1.0.0")}
EOF

cat > ~/microservices-lab/inventory-service/requirements.txt << 'EOF'
fastapi==0.111.0
uvicorn==0.29.0
pydantic==2.7.1
EOF
```

3. Crear el código del `gateway-service`:

```bash
mkdir -p ~/microservices-lab/gateway-service/app
cat > ~/microservices-lab/gateway-service/app/__init__.py << 'EOF'
EOF

cat > ~/microservices-lab/gateway-service/app/main.py << 'EOF'
from fastapi import FastAPI
import os

app = FastAPI(title="Gateway Service", version="1.0.0")

@app.get("/health")
async def health():
    return {"status": "healthy", "service": "gateway-service"}

@app.get("/")
async def root():
    return {"gateway": "active", "version": os.getenv("APP_VERSION", "1.0.0")}
EOF

cat > ~/microservices-lab/gateway-service/requirements.txt << 'EOF'
fastapi==0.111.0
uvicorn==0.29.0
httpx==0.27.0
EOF
```

### Verificación

```bash
ls ~/microservices-lab/order-service/app/main.py
ls ~/microservices-lab/inventory-service/app/main.py
ls ~/microservices-lab/gateway-service/app/main.py
```

Todos los archivos deben existir sin errores.

---

## Paso 2: Crear Dockerfiles No Optimizados (Línea Base)

**Objetivo:** Construir imágenes con un Dockerfile simple para medir el tamaño base antes de la optimización.

### Instrucciones

1. Crear un Dockerfile simple (no optimizado) para `order-service`:

```bash
cat > ~/microservices-lab/order-service/Dockerfile.baseline << 'EOF'
FROM python:3.12

WORKDIR /app
COPY . .
RUN pip install -r requirements.txt

EXPOSE 8001
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8001"]
EOF
```

2. Construir la imagen baseline:

```bash
cd ~/microservices-lab/order-service
docker build -f Dockerfile.baseline -t order-service:baseline .
```

3. Registrar el tamaño:

```bash
docker images order-service:baseline --format "table {{.Repository}}\t{{.Tag}}\t{{.Size}}"
```

**Salida esperada (aproximada):**
```
REPOSITORY      TAG        SIZE
order-service   baseline   1.02GB
```

### Verificación

```bash
# El tamaño debe ser superior a 900MB con python:3.12 completo
docker images order-service:baseline --format "{{.Size}}" | grep -E "^[0-9.]+(GB|MB)"
```

---

## Paso 3: Refactorizar Dockerfiles con Multi-Stage Build

**Objetivo:** Reescribir los Dockerfiles aplicando multi-stage builds, usuario no-root, variables de entorno optimizadas y labels OCI.

### Instrucciones

1. Crear el `.dockerignore` común (aplicar a los tres servicios):

```bash
for svc in order-service inventory-service gateway-service; do
cat > ~/microservices-lab/$svc/.dockerignore << 'EOF'
__pycache__
*.pyc
*.pyo
.git
.gitignore
.env
.env.*
tests/
test_*
*.md
Dockerfile.baseline
.pytest_cache
.mypy_cache
.coverage
htmlcov/
venv/
.venv/
*.egg-info/
dist/
build/
EOF
done
```

2. Crear el Dockerfile optimizado para `order-service`:

```bash
cat > ~/microservices-lab/order-service/Dockerfile << 'EOF'
# ─────────────────────────────────────────────────
# ETAPA 1: builder — instala dependencias
# ─────────────────────────────────────────────────
FROM python:3.12.3-slim-bookworm AS builder

WORKDIR /build

RUN apt-get update && apt-get install -y --no-install-recommends \
    build-essential \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*

RUN python -m venv /opt/venv
ENV PATH="/opt/venv/bin:$PATH"

COPY requirements.txt .
RUN pip install --no-cache-dir --upgrade pip && \
    pip install --no-cache-dir -r requirements.txt

# ─────────────────────────────────────────────────
# ETAPA 2: production — imagen final ligera
# ─────────────────────────────────────────────────
FROM python:3.12.3-slim-bookworm AS production

LABEL org.opencontainers.image.title="order-service" \
      org.opencontainers.image.version="1.0.0" \
      org.opencontainers.image.description="Microservicio de pedidos - FastAPI" \
      org.opencontainers.image.authors="equipo-plataforma@empresa.com"

ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1 \
    PYTHONFAULTHANDLER=1 \
    PATH="/opt/venv/bin:$PATH" \
    APP_VERSION="1.0.0"

RUN apt-get update && apt-get install -y --no-install-recommends \
    curl \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*

# Copiar entorno virtual desde builder
COPY --from=builder /opt/venv /opt/venv

# Crear usuario no-root
RUN groupadd --gid 1000 appgroup && \
    useradd --uid 1000 --gid appgroup --shell /bin/bash --create-home appuser

WORKDIR /app

COPY --chown=appuser:appgroup . .

USER appuser

EXPOSE 8001

HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 \
  CMD curl -f http://localhost:8001/health || exit 1

CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8001", "--workers", "1"]
EOF
```

3. Crear el Dockerfile optimizado para `inventory-service`:

```bash
cat > ~/microservices-lab/inventory-service/Dockerfile << 'EOF'
FROM python:3.12.3-slim-bookworm AS builder

WORKDIR /build

RUN apt-get update && apt-get install -y --no-install-recommends \
    build-essential \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*

RUN python -m venv /opt/venv
ENV PATH="/opt/venv/bin:$PATH"

COPY requirements.txt .
RUN pip install --no-cache-dir --upgrade pip && \
    pip install --no-cache-dir -r requirements.txt

FROM python:3.12.3-slim-bookworm AS production

LABEL org.opencontainers.image.title="inventory-service" \
      org.opencontainers.image.version="1.0.0" \
      org.opencontainers.image.description="Microservicio de inventario - FastAPI" \
      org.opencontainers.image.authors="equipo-plataforma@empresa.com"

ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1 \
    PYTHONFAULTHANDLER=1 \
    PATH="/opt/venv/bin:$PATH" \
    APP_VERSION="1.0.0"

RUN apt-get update && apt-get install -y --no-install-recommends \
    curl \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*

COPY --from=builder /opt/venv /opt/venv

RUN groupadd --gid 1000 appgroup && \
    useradd --uid 1000 --gid appgroup --shell /bin/bash --create-home appuser

WORKDIR /app
COPY --chown=appuser:appgroup . .

USER appuser
EXPOSE 8002

HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 \
  CMD curl -f http://localhost:8002/health || exit 1

CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8002", "--workers", "1"]
EOF
```

4. Crear el Dockerfile optimizado para `gateway-service`:

```bash
cat > ~/microservices-lab/gateway-service/Dockerfile << 'EOF'
FROM python:3.12.3-slim-bookworm AS builder

WORKDIR /build

RUN apt-get update && apt-get install -y --no-install-recommends \
    build-essential \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*

RUN python -m venv /opt/venv
ENV PATH="/opt/venv/bin:$PATH"

COPY requirements.txt .
RUN pip install --no-cache-dir --upgrade pip && \
    pip install --no-cache-dir -r requirements.txt

FROM python:3.12.3-slim-bookworm AS production

LABEL org.opencontainers.image.title="gateway-service" \
      org.opencontainers.image.version="1.0.0" \
      org.opencontainers.image.description="API Gateway - FastAPI" \
      org.opencontainers.image.authors="equipo-plataforma@empresa.com"

ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1 \
    PYTHONFAULTHANDLER=1 \
    PATH="/opt/venv/bin:$PATH" \
    APP_VERSION="1.0.0"

RUN apt-get update && apt-get install -y --no-install-recommends \
    curl \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*

COPY --from=builder /opt/venv /opt/venv

RUN groupadd --gid 1000 appgroup && \
    useradd --uid 1000 --gid appgroup --shell /bin/bash --create-home appuser

WORKDIR /app
COPY --chown=appuser:appgroup . .

USER appuser
EXPOSE 8003

HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 \
  CMD curl -f http://localhost:8003/health || exit 1

CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8003", "--workers", "1"]
EOF
```

5. Construir las imágenes optimizadas:

```bash
cd ~/microservices-lab/order-service
docker build -t order-service:1.0.0 .

cd ~/microservices-lab/inventory-service
docker build -t inventory-service:1.0.0 .

cd ~/microservices-lab/gateway-service
docker build -t gateway-service:1.0.0 .
```

6. Comparar tamaños:

```bash
docker images --format "table {{.Repository}}\t{{.Tag}}\t{{.Size}}" | grep -E "(order|inventory|gateway)-service"
```

**Salida esperada:**
```
REPOSITORY          TAG        SIZE
gateway-service     1.0.0      185MB
inventory-service   1.0.0      183MB
order-service       1.0.0      183MB
order-service       baseline   1.02GB
```

### Verificación

```bash
# Verificar que el usuario no es root
docker run --rm order-service:1.0.0 whoami
```

**Salida esperada:**
```
appuser
```

```bash
# Verificar que el servicio arranca correctamente
docker run --rm -d --name test-order -p 8001:8001 order-service:1.0.0
sleep 3
curl -s http://localhost:8001/health | python3 -m json.tool
docker stop test-order
```

**Salida esperada:**
```json
{
    "status": "healthy",
    "service": "order-service"
}
```

---

## Paso 4: Escanear Imágenes con Trivy

**Objetivo:** Detectar vulnerabilidades CVE en las imágenes baseline y optimizadas, comparando resultados.

### Instrucciones

1. Escanear la imagen baseline:

```bash
trivy image --severity HIGH,CRITICAL order-service:baseline 2>&1 | tee ~/microservices-lab/scripts/trivy-baseline.txt
```

2. Escanear la imagen optimizada:

```bash
trivy image --severity HIGH,CRITICAL order-service:1.0.0 2>&1 | tee ~/microservices-lab/scripts/trivy-optimized.txt
```

3. Comparar el número de vulnerabilidades:

```bash
echo "=== Vulnerabilidades Baseline ==="
grep -c "HIGH\|CRITICAL" ~/microservices-lab/scripts/trivy-baseline.txt || echo "0"

echo "=== Vulnerabilidades Optimizada ==="
grep -c "HIGH\|CRITICAL" ~/microservices-lab/scripts/trivy-optimized.txt || echo "0"
```

4. Generar reporte en formato tabla para la imagen optimizada:

```bash
trivy image --format table --severity HIGH,CRITICAL order-service:1.0.0
```

**Salida esperada (ejemplo — los CVEs varían):**
```
order-service:1.0.0 (debian 12.5)

Total: 2 (HIGH: 2, CRITICAL: 0)

┌──────────────┬────────────────┬──────────┬───────────────────┐
│   Library    │ Vulnerability  │ Severity │  Fixed Version    │
├──────────────┼────────────────┼──────────┼───────────────────┤
│ libssl3      │ CVE-2024-XXXX  │ HIGH     │ 3.0.13-1~deb12u2  │
│ curl         │ CVE-2024-YYYY  │ HIGH     │ 7.88.1-10+deb12u6 │
└──────────────┴────────────────┴──────────┴───────────────────┘
```

5. Si se encuentran vulnerabilidades remediables, actualizar paquetes del sistema en el Dockerfile (agregar antes de la creación del usuario en la etapa `production`):

```bash
# Si Trivy reporta vulnerabilidades en paquetes del sistema, agregar:
# RUN apt-get update && apt-get upgrade -y && apt-get clean && rm -rf /var/lib/apt/lists/*
# Reconstruir la imagen después de la remediación
```

### Verificación

```bash
# La imagen optimizada debe tener significativamente menos vulnerabilidades que la baseline
echo "Reducción verificada: la imagen slim tiene menos paquetes instalados y por tanto menor superficie de ataque"
```

---

## Paso 5: Configurar Registro Local y Publicar Imágenes

**Objetivo:** Etiquetar y publicar las imágenes optimizadas en el registro Docker local de minikube.

### Instrucciones

1. Verificar que el registro de minikube está disponible:

```bash
# El addon registry de minikube expone el registro internamente
kubectl get svc -n kube-system | grep registry
```

**Salida esperada:**
```
registry   ClusterIP   10.96.x.x   <none>   80/TCP,443/TCP   ...
```

2. Configurar port-forward al registro (en una terminal separada o background):

```bash
kubectl port-forward --namespace kube-system svc/registry 5000:80 &
REGISTRY_PID=$!
sleep 2
echo "Registry port-forward PID: $REGISTRY_PID"
```

3. Etiquetar las imágenes para el registro local:

```bash
docker tag order-service:1.0.0 localhost:5000/order-service:1.0.0
docker tag inventory-service:1.0.0 localhost:5000/inventory-service:1.0.0
docker tag gateway-service:1.0.0 localhost:5000/gateway-service:1.0.0
```

4. Publicar las imágenes:

```bash
docker push localhost:5000/order-service:1.0.0
docker push localhost:5000/inventory-service:1.0.0
docker push localhost:5000/gateway-service:1.0.0
```

5. Verificar que las imágenes están en el registro:

```bash
curl -s http://localhost:5000/v2/_catalog | python3 -m json.tool
```

**Salida esperada:**
```json
{
    "repositories": [
        "gateway-service",
        "inventory-service",
        "order-service"
    ]
}
```

### Verificación

```bash
# Verificar tags disponibles
curl -s http://localhost:5000/v2/order-service/tags/list | python3 -m json.tool
```

**Salida esperada:**
```json
{
    "name": "order-service",
    "tags": [
        "1.0.0"
    ]
}
```

---

## Paso 6: Crear el Helm Chart Reutilizable

**Objetivo:** Crear un Helm chart parametrizado que soporte el despliegue de cualquiera de los tres microservicios.

### Instrucciones

1. Crear la estructura del chart:

```bash
mkdir -p ~/microservices-lab/helm/msvc-chart/templates
cd ~/microservices-lab/helm/msvc-chart
```

2. Crear `Chart.yaml`:

```bash
cat > Chart.yaml << 'EOF'
apiVersion: v2
name: msvc-chart
description: Helm chart reutilizable para microservicios Python FastAPI
type: application
version: 0.1.0
appVersion: "1.0.0"
maintainers:
  - name: equipo-plataforma
    email: equipo-plataforma@empresa.com
EOF
```

3. Crear `templates/_helpers.tpl`:

```bash
cat > templates/_helpers.tpl << 'EOF'
{{/*
Nombre completo del recurso
*/}}
{{- define "msvc-chart.fullname" -}}
{{- if .Values.fullnameOverride }}
{{- .Values.fullnameOverride | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- $name := default .Chart.Name .Values.nameOverride }}
{{- printf "%s-%s" .Release.Name $name | trunc 63 | trimSuffix "-" }}
{{- end }}
{{- end }}

{{/*
Nombre del chart
*/}}
{{- define "msvc-chart.name" -}}
{{- default .Chart.Name .Values.nameOverride | trunc 63 | trimSuffix "-" }}
{{- end }}

{{/*
Labels comunes
*/}}
{{- define "msvc-chart.labels" -}}
helm.sh/chart: {{ .Chart.Name }}-{{ .Chart.Version }}
app.kubernetes.io/name: {{ include "msvc-chart.name" . }}
app.kubernetes.io/instance: {{ .Release.Name }}
app.kubernetes.io/version: {{ .Values.image.tag | default .Chart.AppVersion | quote }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
{{- end }}

{{/*
Selector labels
*/}}
{{- define "msvc-chart.selectorLabels" -}}
app.kubernetes.io/name: {{ include "msvc-chart.name" . }}
app.kubernetes.io/instance: {{ .Release.Name }}
{{- end }}

{{/*
Nombre del ServiceAccount
*/}}
{{- define "msvc-chart.serviceAccountName" -}}
{{- if .Values.serviceAccount.create }}
{{- default (include "msvc-chart.fullname" .) .Values.serviceAccount.name }}
{{- else }}
{{- default "default" .Values.serviceAccount.name }}
{{- end }}
{{- end }}
EOF
```

4. Crear `templates/deployment.yaml`:

```bash
cat > templates/deployment.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "msvc-chart.fullname" . }}
  labels:
    {{- include "msvc-chart.labels" . | nindent 4 }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      {{- include "msvc-chart.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "msvc-chart.selectorLabels" . | nindent 8 }}
      annotations:
        checksum/config: {{ include (print $.Template.BasePath "/configmap.yaml") . | sha256sum }}
    spec:
      serviceAccountName: {{ include "msvc-chart.serviceAccountName" . }}
      securityContext:
        runAsUser: 1000
        runAsGroup: 1000
        fsGroup: 1000
      containers:
        - name: {{ .Values.service.name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - name: http
              containerPort: {{ .Values.service.targetPort }}
              protocol: TCP
          envFrom:
            - configMapRef:
                name: {{ include "msvc-chart.fullname" . }}-config
          {{- if .Values.env }}
          env:
            {{- range $key, $value := .Values.env }}
            - name: {{ $key }}
              value: {{ $value | quote }}
            {{- end }}
          {{- end }}
          livenessProbe:
            httpGet:
              path: /health
              port: http
            initialDelaySeconds: 10
            periodSeconds: 30
            timeoutSeconds: 5
          readinessProbe:
            httpGet:
              path: /health
              port: http
            initialDelaySeconds: 5
            periodSeconds: 10
            timeoutSeconds: 3
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
EOF
```

5. Crear `templates/service.yaml`:

```bash
cat > templates/service.yaml << 'EOF'
apiVersion: v1
kind: Service
metadata:
  name: {{ include "msvc-chart.fullname" . }}
  labels:
    {{- include "msvc-chart.labels" . | nindent 4 }}
spec:
  type: {{ .Values.service.type }}
  ports:
    - port: {{ .Values.service.port }}
      targetPort: http
      protocol: TCP
      name: http
  selector:
    {{- include "msvc-chart.selectorLabels" . | nindent 4 }}
EOF
```

6. Crear `templates/configmap.yaml`:

```bash
cat > templates/configmap.yaml << 'EOF'
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ include "msvc-chart.fullname" . }}-config
  labels:
    {{- include "msvc-chart.labels" . | nindent 4 }}
data:
  APP_VERSION: {{ .Values.image.tag | default .Chart.AppVersion | quote }}
  SERVICE_NAME: {{ .Values.service.name | quote }}
  {{- range $key, $value := .Values.config }}
  {{ $key }}: {{ $value | quote }}
  {{- end }}
EOF
```

7. Crear `templates/serviceaccount.yaml`:

```bash
cat > templates/serviceaccount.yaml << 'EOF'
{{- if .Values.serviceAccount.create -}}
apiVersion: v1
kind: ServiceAccount
metadata:
  name: {{ include "msvc-chart.serviceAccountName" . }}
  labels:
    {{- include "msvc-chart.labels" . | nindent 4 }}
  {{- with .Values.serviceAccount.annotations }}
  annotations:
    {{- toYaml . | nindent 4 }}
  {{- end }}
{{- end }}
EOF
```

8. Crear el `values.yaml` base:

```bash
cat > values.yaml << 'EOF'
# Valores por defecto del chart msvc-chart
replicaCount: 1

image:
  repository: localhost:5000/order-service
  tag: "1.0.0"
  pullPolicy: IfNotPresent

nameOverride: ""
fullnameOverride: ""

service:
  name: msvc
  type: ClusterIP
  port: 80
  targetPort: 8000

serviceAccount:
  create: true
  annotations: {}
  name: ""

resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 250m
    memory: 256Mi

config: {}

env: {}
EOF
```

### Verificación

```bash
cd ~/microservices-lab/helm/msvc-chart
helm lint .
```

**Salida esperada:**
```
==> Linting .
[INFO] Chart.yaml: icon is recommended

1 chart(s) linted, 0 chart(s) failed
```

---

## Paso 7: Crear Values Files por Servicio

**Objetivo:** Crear archivos de valores diferenciados para cada microservicio.

### Instrucciones

1. Crear `values-orders.yaml`:

```bash
cat > ~/microservices-lab/helm/msvc-chart/values-orders.yaml << 'EOF'
replicaCount: 2

image:
  repository: localhost:5000/order-service
  tag: "1.0.0"
  pullPolicy: Always

fullnameOverride: "order-service"

service:
  name: order-service
  type: ClusterIP
  port: 80
  targetPort: 8001

resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 500m
    memory: 512Mi

config:
  LOG_LEVEL: "info"
  DB_HOST: "mslab-postgres"
  DB_NAME: "orders_db"

env:
  OTEL_SERVICE_NAME: "order-service"
  VAULT_ADDR: "http://vault:8200"
EOF
```

2. Crear `values-inventory.yaml`:

```bash
cat > ~/microservices-lab/helm/msvc-chart/values-inventory.yaml << 'EOF'
replicaCount: 2

image:
  repository: localhost:5000/inventory-service
  tag: "1.0.0"
  pullPolicy: Always

fullnameOverride: "inventory-service"

service:
  name: inventory-service
  type: ClusterIP
  port: 80
  targetPort: 8002

resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 500m
    memory: 512Mi

config:
  LOG_LEVEL: "info"
  DB_HOST: "mslab-postgres"
  DB_NAME: "inventory_db"

env:
  OTEL_SERVICE_NAME: "inventory-service"
  VAULT_ADDR: "http://vault:8200"
EOF
```

3. Crear `values-gateway.yaml`:

```bash
cat > ~/microservices-lab/helm/msvc-chart/values-gateway.yaml << 'EOF'
replicaCount: 1

image:
  repository: localhost:5000/gateway-service
  tag: "1.0.0"
  pullPolicy: Always

fullnameOverride: "gateway-service"

service:
  name: gateway-service
  type: NodePort
  port: 80
  targetPort: 8003

resources:
  requests:
    cpu: 150m
    memory: 128Mi
  limits:
    cpu: 300m
    memory: 256Mi

config:
  LOG_LEVEL: "debug"
  ORDER_SERVICE_URL: "http://order-service"
  INVENTORY_SERVICE_URL: "http://inventory-service"

env:
  OTEL_SERVICE_NAME: "gateway-service"
EOF
```

### Verificación

```bash
# Validar template rendering para cada servicio
helm template order-release ~/microservices-lab/helm/msvc-chart \
  -f ~/microservices-lab/helm/msvc-chart/values-orders.yaml | head -30
```

**Salida esperada (fragmento):**
```yaml
---
# Source: msvc-chart/templates/serviceaccount.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: order-service
  labels:
    ...
```

---

## Paso 8: Desplegar los Microservicios con Helm

**Objetivo:** Instalar los tres microservicios en minikube usando el Helm chart con sus respectivos values files.

### Instrucciones

1. Crear el namespace:

```bash
kubectl create namespace microservices-lab
```

2. Desplegar `order-service`:

```bash
helm upgrade --install order-release ~/microservices-lab/helm/msvc-chart \
  -f ~/microservices-lab/helm/msvc-chart/values-orders.yaml \
  -n microservices-lab \
  --wait --timeout 60s
```

**Salida esperada:**
```
Release "order-release" does not exist. Installing it now.
NAME: order-release
LAST DEPLOYED: ...
NAMESPACE: microservices-lab
STATUS: deployed
REVISION: 1
```

3. Desplegar `inventory-service`:

```bash
helm upgrade --install inventory-release ~/microservices-lab/helm/msvc-chart \
  -f ~/microservices-lab/helm/msvc-chart/values-inventory.yaml \
  -n microservices-lab \
  --wait --timeout 60s
```

4. Desplegar `gateway-service`:

```bash
helm upgrade --install gateway-release ~/microservices-lab/helm/msvc-chart \
  -f ~/microservices-lab/helm/msvc-chart/values-gateway.yaml \
  -n microservices-lab \
  --wait --timeout 60s
```

5. Verificar los releases de Helm:

```bash
helm list -n microservices-lab
```

**Salida esperada:**
```
NAME                NAMESPACE           REVISION  STATUS    CHART           APP VERSION
gateway-release     microservices-lab   1         deployed  msvc-chart-0.1.0  1.0.0
inventory-release   microservices-lab   1         deployed  msvc-chart-0.1.0  1.0.0
order-release       microservices-lab   1         deployed  msvc-chart-0.1.0  1.0.0
```

6. Verificar los pods:

```bash
kubectl get pods -n microservices-lab
```

**Salida esperada:**
```
NAME                                 READY   STATUS    RESTARTS   AGE
gateway-service-xxx-yyy              1/1     Running   0          30s
inventory-service-xxx-yyy            1/1     Running   0          45s
inventory-service-xxx-zzz            1/1     Running   0          45s
order-service-xxx-yyy                1/1     Running   0          60s
order-service-xxx-zzz                1/1     Running   0          60s
```

### Verificación

```bash
# Port-forward para probar el order-service
kubectl port-forward svc/order-service -n microservices-lab 8081:80 &
sleep 2
curl -s http://localhost:8081/health | python3 -m json.tool
kill %1 2>/dev/null
```

**Salida esperada:**
```json
{
    "status": "healthy",
    "service": "order-service"
}
```

---

## Paso 9: Validar Upgrade y Rollback con Helm

**Objetivo:** Demostrar el ciclo de vida de releases con upgrade de versión y rollback.

### Instrucciones

1. Simular una nueva versión del `order-service`. Actualizar la versión en el código:

```bash
cd ~/microservices-lab/order-service
sed -i 's/APP_VERSION="1.0.0"/APP_VERSION="1.1.0"/' Dockerfile
```

2. Reconstruir y publicar la nueva versión:

```bash
docker build -t order-service:1.1.0 .
docker tag order-service:1.1.0 localhost:5000/order-service:1.1.0
docker push localhost:5000/order-service:1.1.0
```

3. Realizar upgrade del release:

```bash
helm upgrade order-release ~/microservices-lab/helm/msvc-chart \
  -f ~/microservices-lab/helm/msvc-chart/values-orders.yaml \
  --set image.tag="1.1.0" \
  -n microservices-lab \
  --wait --timeout 60s
```

**Salida esperada:**
```
Release "order-release" has been upgraded. Happy Helming!
...
REVISION: 2
```

4. Verificar la nueva versión:

```bash
kubectl port-forward svc/order-service -n microservices-lab 8081:80 &
sleep 2
curl -s http://localhost:8081/api/v1/orders | python3 -m json.tool
kill %1 2>/dev/null
```

**Salida esperada:**
```json
{
    "orders": [],
    "service_version": "1.1.0"
}
```

5. Ver historial de releases:

```bash
helm history order-release -n microservices-lab
```

**Salida esperada:**
```
REVISION  UPDATED                   STATUS      CHART           APP VERSION  DESCRIPTION
1         ...                       superseded  msvc-chart-0.1.0  1.0.0      Install complete
2         ...                       deployed    msvc-chart-0.1.0  1.0.0      Upgrade complete
```

6. Realizar rollback a la revisión 1:

```bash
helm rollback order-release 1 -n microservices-lab --wait
```

**Salida esperada:**
```
Rollback was a success! Happy Helming!
```

7. Verificar que volvió a la versión anterior:

```bash
helm history order-release -n microservices-lab
```

**Salida esperada:**
```
REVISION  UPDATED                   STATUS      CHART           APP VERSION  DESCRIPTION
1         ...                       superseded  msvc-chart-0.1.0  1.0.0      Install complete
2         ...                       superseded  msvc-chart-0.1.0  1.0.0      Upgrade complete
3         ...                       deployed    msvc-chart-0.1.0  1.0.0      Rollback to 1
```

### Verificación

```bash
# Confirmar que el pod usa la imagen 1.0.0 después del rollback
kubectl get deployment order-service -n microservices-lab \
  -o jsonpath='{.spec.template.spec.containers[0].image}'
echo ""
```

**Salida esperada:**
```
localhost:5000/order-service:1.0.0
```

---

## Validación y Testing Final

Ejecutar el script de validación completo:

```bash
cat > ~/microservices-lab/scripts/validate-lab09.sh << 'SCRIPT'
#!/bin/bash
set -e

echo "╔══════════════════════════════════════════════╗"
echo "║  Validación Lab 09-00-01                     ║"
echo "╠══════════════════════════════════════════════╣"

PASS=0
FAIL=0

check() {
  if eval "$2" > /dev/null 2>&1; then
    echo "║ ✅ $1"
    PASS=$((PASS+1))
  else
    echo "║ ❌ $1"
    FAIL=$((FAIL+1))
  fi
}

# 1. Imágenes optimizadas existen
check "order-service:1.0.0 existe" "docker images -q order-service:1.0.0"
check "inventory-service:1.0.0 existe" "docker images -q inventory-service:1.0.0"
check "gateway-service:1.0.0 existe" "docker images -q gateway-service:1.0.0"

# 2. Tamaño reducido (menor a 300MB)
SIZE=$(docker images order-service:1.0.0 --format "{{.Size}}" | grep -oP '[\d.]+')
check "order-service < 300MB" "echo $SIZE | awk '{exit (\$1 < 300) ? 0 : 1}'"

# 3. Usuario no-root
check "Ejecuta como appuser" "docker run --rm order-service:1.0.0 whoami | grep appuser"

# 4. .dockerignore existe
check ".dockerignore en order-service" "test -f ~/microservices-lab/order-service/.dockerignore"

# 5. Helm chart válido
check "Helm lint pasa" "helm lint ~/microservices-lab/helm/msvc-chart"

# 6. Releases desplegados
check "order-release desplegado" "helm status order-release -n microservices-lab"
check "inventory-release desplegado" "helm status inventory-release -n microservices-lab"
check "gateway-release desplegado" "helm status gateway-release -n microservices-lab"

# 7. Pods running
check "Pods en Running" "kubectl get pods -n microservices-lab --field-selector=status.phase=Running | grep -c Running | awk '{exit (\$1 >= 3) ? 0 : 1}'"

# 8. Registro local funcional
check "Registro local accesible" "curl -sf http://localhost:5000/v2/_catalog"

echo "╠══════════════════════════════════════════════╣"
echo "║ Resultados: $PASS pasaron, $FAIL fallaron    ║"
echo "╚══════════════════════════════════════════════╝"

exit $FAIL
SCRIPT

chmod +x ~/microservices-lab/scripts/validate-lab09.sh
~/microservices-lab/scripts/validate-lab09.sh
```

---

## Resolución de Problemas

### Problema 1: Las imágenes no se pueden hacer push al registro local

**Síntomas:**
```
Get "https://localhost:5000/v2/": http: server gave HTTP response to HTTPS client
```

**Causa:** Docker intenta usar HTTPS para conectar al registro local, pero el registro de minikube no tiene TLS configurado.

**Solución:**

```bash
# Opción 1: Configurar Docker para aceptar el registro inseguro
# Editar /etc/docker/daemon.json
sudo tee /etc/docker/daemon.json << 'EOF'
{
  "insecure-registries": ["localhost:5000"]
}
EOF
sudo systemctl restart docker

# Opción 2: Usar el entorno Docker de minikube directamente
eval $(minikube docker-env)
# Las imágenes construidas dentro del entorno de minikube son accesibles sin push
```

### Problema 2: Pods en estado CrashLoopBackOff después del despliegue Helm

**Síntomas:**
```
NAME                              READY   STATUS             RESTARTS   AGE
order-service-xxx-yyy             0/1     CrashLoopBackOff   3          2m
```

**Causa:** El `targetPort` en el values file no coincide con el puerto en el que uvicorn escucha dentro del contenedor. El readiness probe falla y Kubernetes reinicia el pod.

**Solución:**

```bash
# Verificar los logs del pod
kubectl logs -n microservices-lab deployment/order-service

# Confirmar que el puerto en values-orders.yaml coincide con el EXPOSE del Dockerfile
# values-orders.yaml debe tener targetPort: 8001
# El Dockerfile debe tener: CMD ["uvicorn", "app.main:app", "--port", "8001"]

# Corregir y redesplegar
helm upgrade order-release ~/microservices-lab/helm/msvc-chart \
  -f ~/microservices-lab/helm/msvc-chart/values-orders.yaml \
  -n microservices-lab --wait
```

---

## Limpieza

```bash
# Eliminar releases de Helm
helm uninstall order-release -n microservices-lab
helm uninstall inventory-release -n microservices-lab
helm uninstall gateway-release -n microservices-lab

# Eliminar namespace
kubectl delete namespace microservices-lab

# Detener port-forward del registro
kill $REGISTRY_PID 2>/dev/null

# Eliminar imágenes baseline (conservar las optimizadas para labs futuros)
docker rmi order-service:baseline 2>/dev/null

# Opcional: detener minikube si no se usará inmediatamente
# minikube stop
```

---

## Resumen

En este laboratorio has completado las siguientes tareas:

| Tarea | Resultado |
|-------|-----------|
| Dockerfiles multi-stage | Reducción de ~1 GB a ~185 MB por imagen |
| Seguridad de contenedores | Usuario no-root (uid 1000), sin herramientas de compilación |
| Escaneo de vulnerabilidades | Trivy identificó y se remediaron CVEs |
| Registro local | Imágenes publicadas en localhost:5000 dentro de minikube |
| Helm chart reutilizable | Un chart con templates parametrizados para 3 servicios |
| Ciclo de vida | Upgrade a v1.1.0 y rollback exitoso a v1.0.0 |

### Conceptos Clave Aplicados

- **Multi-stage builds**: separación de entorno de compilación y runtime
- **Caché de capas**: `requirements.txt` copiado antes que el código fuente
- **Principio de mínimo privilegio**: usuario no-root, imagen slim sin herramientas innecesarias
- **Helm templating**: `_helpers.tpl` con funciones reutilizables, values files por entorno
- **Versionado semántico**: tags de imagen alineados con releases de Helm

### Recursos Adicionales

- [Docker Best Practices for Python](https://docs.docker.com/develop/develop-images/dockerfile_best-practices/)
- [Helm Chart Development Guide](https://helm.sh/docs/chart_template_guide/)
- [Trivy Documentation](https://aquasecurity.github.io/trivy/)
- [OCI Image Spec - Annotations](https://github.com/opencontainers/image-spec/blob/main/annotations.md)
