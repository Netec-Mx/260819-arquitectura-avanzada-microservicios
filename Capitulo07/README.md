# Integrar Vault con Servicios Python para Gestión de Secretos

## Metadata

| Campo | Valor |
|-------|-------|
| **Duración** | 66 minutos |
| **Complejidad** | Alta |
| **Nivel Bloom** | Aplicar |
| **Tecnologías** | HashiCorp Vault 1.17.1, hvac 2.3.0, Vault Kubernetes Auth, KV v2, FastAPI 0.111.0 |

## Descripción General

En este laboratorio se despliega HashiCorp Vault en un clúster minikube existente y se integra con tres microservicios FastAPI (orders, inventory, users) para gestionar secretos de forma centralizada. Se configuran políticas de acceso granular, autenticación mediante ServiceAccounts de Kubernetes, renovación automática de tokens y rotación de secretos sin downtime. Al finalizar, cada servicio accede exclusivamente a sus propios secretos con trazabilidad completa mediante audit logs.

## Objetivos de Aprendizaje

- [ ] Desplegar HashiCorp Vault 1.17.1 en Kubernetes usando Helm y configurar el motor KV v2
- [ ] Configurar autenticación Kubernetes en Vault vinculando ServiceAccounts con políticas de acceso mínimo
- [ ] Implementar lectura de secretos desde Vault en microservicios FastAPI usando hvac 2.3.0
- [ ] Demostrar rotación de secretos sin reinicio de servicios mediante background tasks de renovación
- [ ] Auditar accesos a secretos mediante el audit log de Vault

## Prerrequisitos

### Conocimientos Requeridos

- Administración básica de Kubernetes (namespaces, ServiceAccounts, RBAC)
- Experiencia con FastAPI y programación asíncrona en Python
- Conceptos fundamentales de PKI, tokens y políticas de acceso
- Lab 06-00-01 completado con tres microservicios desplegados en minikube

### Acceso y Herramientas

| Herramienta | Versión | Verificación |
|-------------|---------|--------------|
| minikube | 1.33+ | `minikube status` |
| kubectl | 1.30+ | `kubectl version --client` |
| Helm | 3.15.2 | `helm version` |
| Vault CLI | 1.17.1 | `vault version` |
| Python | 3.12.3 | `python3 --version` |
| hvac | 2.3.0 | `pip show hvac` |

## Entorno del Laboratorio

### Estructura de Directorios

```
/home/student/msvc-labs/vault/
├── helm/
│   └── vault-values.yaml
├── policies/
│   ├── orders-policy.hcl
│   ├── inventory-policy.hcl
│   └── users-policy.hcl
├── k8s/
│   ├── serviceaccounts.yaml
│   └── deployments/
│       ├── orders-deployment.yaml
│       ├── inventory-deployment.yaml
│       └── users-deployment.yaml
├── app/
│   ├── vault_client.py
│   └── config.py
└── scripts/
    ├── setup-vault.sh
    └── rotate-secrets.sh
```

### Preparación Inicial

```bash
# Crear estructura de directorios
mkdir -p /home/student/msvc-labs/vault/{helm,policies,k8s/deployments,app,scripts}
cd /home/student/msvc-labs/vault

# Verificar que minikube está corriendo con los servicios del lab anterior
minikube status
kubectl get pods -n default

# Añadir repositorio Helm de HashiCorp (si no existe)
helm repo add hashicorp https://helm.releases.hashicorp.com
helm repo update
```

---

## Paso 1: Desplegar HashiCorp Vault en Kubernetes

**Objetivo:** Instalar Vault 1.17.1 en el namespace `vault` del clúster minikube usando Helm con configuración de desarrollo habilitada.

### Instrucciones

1. Crear el namespace para Vault:

```bash
kubectl create namespace vault
```

2. Crear el archivo de valores personalizados para Helm:

```bash
cat > /home/student/msvc-labs/vault/helm/vault-values.yaml << 'EOF'
server:
  image:
    repository: hashicorp/vault
    tag: "1.17.1"
  dev:
    enabled: true
    devRootToken: "root-token-msvc-lab"
  resources:
    requests:
      memory: "256Mi"
      cpu: "250m"
    limits:
      memory: "512Mi"
      cpu: "500m"
  service:
    type: NodePort
    nodePort: 30820
  auditStorage:
    enabled: true
    size: 1Gi

ui:
  enabled: true

injector:
  enabled: false
EOF
```

3. Instalar Vault con Helm:

```bash
helm install vault hashicorp/vault \
  --namespace vault \
  --values /home/student/msvc-labs/vault/helm/vault-values.yaml \
  --version 0.28.0
```

4. Esperar a que el pod esté listo:

```bash
kubectl wait --for=condition=Ready pod/vault-0 \
  --namespace vault --timeout=120s
```

5. Configurar acceso al CLI de Vault:

```bash
# Obtener la URL de Vault
export VAULT_ADDR="http://$(minikube ip):30820"
export VAULT_TOKEN="root-token-msvc-lab"

# Guardar variables para uso posterior
echo "export VAULT_ADDR=\"${VAULT_ADDR}\"" >> ~/.bashrc
echo "export VAULT_TOKEN=\"${VAULT_TOKEN}\"" >> ~/.bashrc
```

6. Verificar la conectividad:

```bash
vault status
```

### Salida Esperada

```
Key             Value
---             -----
Seal Type       shamir
Initialized     true
Sealed          false
Total Shares    1
Threshold       1
Version         1.17.1
Storage Type    inmem
Cluster Name    vault-cluster-...
```

### Verificación

```bash
vault secrets list
# Debe mostrar cubbyhole/, identity/, secret/, sys/
```

---

## Paso 2: Configurar el Motor KV v2 y Poblar Secretos

**Objetivo:** Habilitar el motor de secretos KV versión 2 y crear secretos específicos para cada microservicio bajo rutas separadas.

### Instrucciones

1. El motor KV v2 ya está habilitado en `secret/` en modo dev. Verificar:

```bash
vault secrets list -format=json | jq '."secret/"'
```

2. Crear secretos para el servicio de órdenes:

```bash
vault kv put secret/orders \
  database_url="postgresql://orders_user:0rd3rs_s3cr3t@mslab-postgres:5432/orders_db" \
  api_key="orders-api-key-v1-abc123" \
  redis_url="redis://mslab-redis:6379/0" \
  jwt_secret="orders-jwt-hmac-secret-2024"
```

3. Crear secretos para el servicio de inventario:

```bash
vault kv put secret/inventory \
  database_url="postgresql://inventory_user:1nv_s3cr3t@mslab-postgres:5432/inventory_db" \
  api_key="inventory-api-key-v1-def456" \
  redis_url="redis://mslab-redis:6379/1" \
  warehouse_token="wh-external-token-2024"
```

4. Crear secretos para el servicio de usuarios:

```bash
vault kv put secret/users \
  database_url="postgresql://users_user:us3rs_s3cr3t@mslab-postgres:5432/users_db" \
  api_key="users-api-key-v1-ghi789" \
  smtp_password="smtp-mail-password-2024" \
  encryption_key="aes256-user-data-key-2024"
```

5. Verificar que los secretos se almacenaron correctamente:

```bash
vault kv get secret/orders
vault kv get -field=database_url secret/inventory
vault kv get -format=json secret/users | jq '.data.data'
```

### Salida Esperada

```json
{
  "api_key": "users-api-key-v1-ghi789",
  "database_url": "postgresql://users_user:us3rs_s3cr3t@mslab-postgres:5432/users_db",
  "encryption_key": "aes256-user-data-key-2024",
  "smtp_password": "smtp-mail-password-2024"
}
```

### Verificación

```bash
# Confirmar que existen las tres rutas
vault kv list secret/
# Debe mostrar: orders, inventory, users (o las claves directamente)
vault kv metadata get secret/orders | grep -i "current_version"
```

---

## Paso 3: Crear Políticas de Acceso Granular

**Objetivo:** Definir políticas HCL que restrinjan el acceso de cada microservicio exclusivamente a su propia ruta de secretos.

### Instrucciones

1. Crear la política para el servicio de órdenes:

```bash
cat > /home/student/msvc-labs/vault/policies/orders-policy.hcl << 'EOF'
# Política: orders-service
# Permite lectura de secretos bajo secret/data/orders únicamente
path "secret/data/orders" {
  capabilities = ["read"]
}

# Permitir listar metadata (necesario para verificar versiones)
path "secret/metadata/orders" {
  capabilities = ["read", "list"]
}

# Denegar explícitamente acceso a otros servicios
path "secret/data/inventory" {
  capabilities = ["deny"]
}

path "secret/data/users" {
  capabilities = ["deny"]
}

# Permitir renovación del propio token
path "auth/token/renew-self" {
  capabilities = ["update"]
}

path "auth/token/lookup-self" {
  capabilities = ["read"]
}
EOF
```

2. Crear la política para el servicio de inventario:

```bash
cat > /home/student/msvc-labs/vault/policies/inventory-policy.hcl << 'EOF'
# Política: inventory-service
path "secret/data/inventory" {
  capabilities = ["read"]
}

path "secret/metadata/inventory" {
  capabilities = ["read", "list"]
}

path "secret/data/orders" {
  capabilities = ["deny"]
}

path "secret/data/users" {
  capabilities = ["deny"]
}

path "auth/token/renew-self" {
  capabilities = ["update"]
}

path "auth/token/lookup-self" {
  capabilities = ["read"]
}
EOF
```

3. Crear la política para el servicio de usuarios:

```bash
cat > /home/student/msvc-labs/vault/policies/users-policy.hcl << 'EOF'
# Política: users-service
path "secret/data/users" {
  capabilities = ["read"]
}

path "secret/metadata/users" {
  capabilities = ["read", "list"]
}

path "secret/data/orders" {
  capabilities = ["deny"]
}

path "secret/data/inventory" {
  capabilities = ["deny"]
}

path "auth/token/renew-self" {
  capabilities = ["update"]
}

path "auth/token/lookup-self" {
  capabilities = ["read"]
}
EOF
```

4. Cargar las políticas en Vault:

```bash
vault policy write orders-policy \
  /home/student/msvc-labs/vault/policies/orders-policy.hcl

vault policy write inventory-policy \
  /home/student/msvc-labs/vault/policies/inventory-policy.hcl

vault policy write users-policy \
  /home/student/msvc-labs/vault/policies/users-policy.hcl
```

5. Verificar las políticas cargadas:

```bash
vault policy list
vault policy read orders-policy
```

### Salida Esperada

```
default
inventory-policy
orders-policy
root
users-policy
```

### Verificación

```bash
# Crear un token de prueba con la política de orders y verificar aislamiento
ORDERS_TOKEN=$(vault token create -policy=orders-policy -format=json | jq -r '.auth.client_token')

# Debe funcionar: leer secretos de orders
VAULT_TOKEN=$ORDERS_TOKEN vault kv get secret/orders

# Debe fallar: leer secretos de inventory
VAULT_TOKEN=$ORDERS_TOKEN vault kv get secret/inventory 2>&1 | grep -i "permission denied"
echo "Aislamiento verificado correctamente"
```

---

## Paso 4: Configurar Autenticación Kubernetes

**Objetivo:** Habilitar el método de autenticación Kubernetes en Vault y crear roles vinculados a ServiceAccounts específicos de cada microservicio.

### Instrucciones

1. Crear ServiceAccounts dedicados para cada microservicio:

```bash
cat > /home/student/msvc-labs/vault/k8s/serviceaccounts.yaml << 'EOF'
apiVersion: v1
kind: ServiceAccount
metadata:
  name: orders-service-sa
  namespace: default
  labels:
    app: orders-service
    vault-access: "true"
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: inventory-service-sa
  namespace: default
  labels:
    app: inventory-service
    vault-access: "true"
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: users-service-sa
  namespace: default
  labels:
    app: users-service
    vault-access: "true"
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: vault-auth-delegator
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:auth-delegator
subjects:
  - kind: ServiceAccount
    name: vault
    namespace: vault
EOF

kubectl apply -f /home/student/msvc-labs/vault/k8s/serviceaccounts.yaml
```

2. Habilitar el método de autenticación Kubernetes en Vault:

```bash
vault auth enable kubernetes
```

3. Configurar el método de autenticación con los datos del clúster:

```bash
# Obtener la dirección del API server de Kubernetes
K8S_HOST=$(kubectl config view --minify -o jsonpath='{.clusters[0].cluster.server}')

# Obtener el certificado CA del clúster
K8S_CA_CERT=$(kubectl config view --raw --minify -o jsonpath='{.clusters[0].cluster.certificate-authority-data}' | base64 -d)

# Configurar Vault para comunicarse con el API server de Kubernetes
vault write auth/kubernetes/config \
  kubernetes_host="${K8S_HOST}" \
  kubernetes_ca_cert="${K8S_CA_CERT}" \
  disable_local_ca_jwt=true
```

4. Crear roles Vault vinculados a cada ServiceAccount:

```bash
# Rol para orders-service
vault write auth/kubernetes/role/orders-role \
  bound_service_account_names=orders-service-sa \
  bound_service_account_namespaces=default \
  policies=orders-policy \
  ttl=1h \
  max_ttl=4h

# Rol para inventory-service
vault write auth/kubernetes/role/inventory-role \
  bound_service_account_names=inventory-service-sa \
  bound_service_account_namespaces=default \
  policies=inventory-policy \
  ttl=1h \
  max_ttl=4h

# Rol para users-service
vault write auth/kubernetes/role/users-role \
  bound_service_account_names=users-service-sa \
  bound_service_account_namespaces=default \
  policies=users-policy \
  ttl=1h \
  max_ttl=4h
```

5. Verificar la configuración de autenticación:

```bash
vault read auth/kubernetes/role/orders-role
```

### Salida Esperada

```
Key                                 Value
---                                 -----
bound_service_account_names         [orders-service-sa]
bound_service_account_namespaces    [default]
max_ttl                             4h
policies                            [orders-policy]
ttl                                 1h
```

### Verificación

```bash
vault auth list
# Debe incluir kubernetes/ en la lista
vault read auth/kubernetes/config | grep kubernetes_host
```

---

## Paso 5: Habilitar Audit Log en Vault

**Objetivo:** Activar el registro de auditoría de Vault para rastrear todos los accesos a secretos.

### Instrucciones

1. Habilitar el backend de auditoría tipo archivo:

```bash
# Habilitar audit log dentro del pod de Vault
kubectl exec -n vault vault-0 -- vault audit enable file \
  file_path=/vault/audit/vault-audit.log

# Alternativa: habilitar desde el CLI del host (si el anterior falla por permisos)
vault audit enable file file_path=stdout
```

> **Nota:** En modo dev, usar `stdout` como destino del audit log es más práctico para verificación. En producción se usaría un archivo persistente o un backend como syslog.

2. Verificar que el audit log está habilitado:

```bash
vault audit list
```

3. Generar una entrada de auditoría de prueba:

```bash
vault kv get secret/orders
```

4. Verificar la entrada en los logs:

```bash
kubectl logs -n vault vault-0 | grep "secret/data/orders" | tail -1 | jq '.request.path'
```

### Salida Esperada

```
"secret/data/orders"
```

### Verificación

```bash
vault audit list -format=json | jq 'keys'
# Debe mostrar al menos un backend habilitado
```

---

## Paso 6: Implementar Cliente Vault en Python con hvac

**Objetivo:** Crear un módulo Python reutilizable que encapsule la comunicación con Vault, incluyendo autenticación Kubernetes, lectura de secretos y renovación automática de tokens.

### Instrucciones

1. Crear el módulo del cliente Vault:

```bash
cat > /home/student/msvc-labs/vault/app/vault_client.py << 'EOF'
"""
Cliente Vault para microservicios FastAPI.
Implementa autenticación Kubernetes, lectura de secretos KV v2
y renovación automática de tokens en background.
"""

import os
import time
import logging
import asyncio
from typing import Any

import hvac
from hvac.exceptions import VaultError, InvalidPath

logger = logging.getLogger(__name__)


class VaultClient:
    """
    Cliente para HashiCorp Vault con soporte para autenticación Kubernetes
    y renovación automática de tokens.
    """

    def __init__(
        self,
        vault_addr: str | None = None,
        vault_role: str | None = None,
        secret_path: str | None = None,
        token: str | None = None,
    ):
        self.vault_addr = vault_addr or os.environ.get(
            "VAULT_ADDR", "http://vault.vault.svc.cluster.local:8200"
        )
        self.vault_role = vault_role or os.environ.get("VAULT_ROLE", "")
        self.secret_path = secret_path or os.environ.get("VAULT_SECRET_PATH", "")
        self._token = token or os.environ.get("VAULT_TOKEN", "")
        self._client: hvac.Client | None = None
        self._secrets_cache: dict[str, Any] = {}
        self._cache_timestamp: float = 0
        self._cache_ttl: int = 60  # Segundos antes de refrescar caché
        self._renewal_task: asyncio.Task | None = None

    def initialize(self) -> None:
        """Inicializa el cliente Vault y autentica."""
        self._client = hvac.Client(url=self.vault_addr)

        if self._token:
            # Autenticación directa con token (desarrollo/testing)
            self._client.token = self._token
            logger.info("Vault: autenticado con token directo")
        else:
            # Autenticación Kubernetes (producción)
            self._authenticate_kubernetes()

        if not self._client.is_authenticated():
            raise VaultError("No se pudo autenticar con Vault")

        logger.info(
            "Vault: cliente inicializado correctamente en %s", self.vault_addr
        )

    def _authenticate_kubernetes(self) -> None:
        """Autentica usando el JWT del ServiceAccount de Kubernetes."""
        jwt_path = "/var/run/secrets/kubernetes.io/serviceaccount/token"
        try:
            with open(jwt_path, "r") as f:
                jwt_token = f.read().strip()

            response = self._client.auth.kubernetes.login(
                role=self.vault_role,
                jwt=jwt_token,
            )
            self._client.token = response["auth"]["client_token"]
            logger.info(
                "Vault: autenticado via Kubernetes auth (role=%s)",
                self.vault_role,
            )
        except FileNotFoundError:
            raise VaultError(
                f"JWT de ServiceAccount no encontrado en {jwt_path}. "
                "¿El pod tiene un ServiceAccount asignado?"
            )
        except Exception as e:
            raise VaultError(f"Error en autenticación Kubernetes: {e}")

    def read_secrets(self, path: str | None = None) -> dict[str, Any]:
        """
        Lee secretos desde Vault KV v2.
        Usa caché local para reducir llamadas al servidor.
        """
        target_path = path or self.secret_path
        now = time.time()

        # Retornar caché si es válido
        if (
            self._secrets_cache
            and (now - self._cache_timestamp) < self._cache_ttl
        ):
            return self._secrets_cache

        try:
            response = self._client.secrets.kv.v2.read_secret_version(
                path=target_path,
                mount_point="secret",
            )
            self._secrets_cache = response["data"]["data"]
            self._cache_timestamp = now
            logger.info("Vault: secretos leídos desde '%s'", target_path)
            return self._secrets_cache

        except InvalidPath:
            logger.error("Vault: ruta de secretos no encontrada: %s", target_path)
            raise
        except VaultError as e:
            logger.error("Vault: error al leer secretos: %s", e)
            # Retornar caché stale si existe
            if self._secrets_cache:
                logger.warning("Vault: usando caché stale para '%s'", target_path)
                return self._secrets_cache
            raise

    def get_secret(self, key: str, default: Any = None) -> Any:
        """Obtiene un secreto específico por clave."""
        secrets = self.read_secrets()
        return secrets.get(key, default)

    def invalidate_cache(self) -> None:
        """Invalida el caché forzando la próxima lectura desde Vault."""
        self._secrets_cache = {}
        self._cache_timestamp = 0
        logger.info("Vault: caché invalidado")

    async def start_token_renewal(self, interval: int = 1800) -> None:
        """
        Inicia la tarea de renovación automática de tokens.
        Se ejecuta como background task de FastAPI.
        """
        self._renewal_task = asyncio.create_task(
            self._renewal_loop(interval)
        )
        logger.info(
            "Vault: renovación automática de token iniciada (intervalo=%ds)",
            interval,
        )

    async def _renewal_loop(self, interval: int) -> None:
        """Loop de renovación que renueva el token periódicamente."""
        while True:
            await asyncio.sleep(interval)
            try:
                self._client.auth.token.renew_self()
                logger.info("Vault: token renovado exitosamente")
                # Aprovechar para refrescar secretos
                self.invalidate_cache()
                self.read_secrets()
                logger.info("Vault: secretos refrescados tras renovación")
            except VaultError as e:
                logger.error("Vault: error al renovar token: %s", e)
                # Intentar re-autenticar
                try:
                    self._authenticate_kubernetes()
                    logger.info("Vault: re-autenticación exitosa")
                except VaultError:
                    logger.critical(
                        "Vault: no se pudo re-autenticar. "
                        "Servicios usando caché stale."
                    )

    async def stop_token_renewal(self) -> None:
        """Detiene la tarea de renovación."""
        if self._renewal_task:
            self._renewal_task.cancel()
            try:
                await self._renewal_task
            except asyncio.CancelledError:
                pass
            logger.info("Vault: renovación automática detenida")

    def close(self) -> None:
        """Cierra la conexión con Vault."""
        if self._client:
            self._client.adapter.close()
            logger.info("Vault: conexión cerrada")
EOF
```

2. Crear el módulo de configuración que usa el cliente Vault:

```bash
cat > /home/student/msvc-labs/vault/app/config.py << 'EOF'
"""
Módulo de configuración que integra Vault con FastAPI.
Proporciona un objeto de configuración que se actualiza dinámicamente.
"""

import os
import logging
from dataclasses import dataclass, field
from typing import Any

from vault_client import VaultClient

logger = logging.getLogger(__name__)


@dataclass
class ServiceConfig:
    """Configuración del servicio obtenida desde Vault."""
    database_url: str = ""
    api_key: str = ""
    redis_url: str = ""
    extra: dict[str, Any] = field(default_factory=dict)

    @classmethod
    def from_vault(cls, vault_client: VaultClient) -> "ServiceConfig":
        """Construye la configuración leyendo secretos desde Vault."""
        secrets = vault_client.read_secrets()
        known_keys = {"database_url", "api_key", "redis_url"}
        extra = {k: v for k, v in secrets.items() if k not in known_keys}

        return cls(
            database_url=secrets.get("database_url", ""),
            api_key=secrets.get("api_key", ""),
            redis_url=secrets.get("redis_url", ""),
            extra=extra,
        )

    def refresh(self, vault_client: VaultClient) -> None:
        """Refresca la configuración desde Vault."""
        vault_client.invalidate_cache()
        secrets = vault_client.read_secrets()
        self.database_url = secrets.get("database_url", self.database_url)
        self.api_key = secrets.get("api_key", self.api_key)
        self.redis_url = secrets.get("redis_url", self.redis_url)
        known_keys = {"database_url", "api_key", "redis_url"}
        self.extra = {k: v for k, v in secrets.items() if k not in known_keys}
        logger.info("Configuración refrescada desde Vault")


def create_vault_client() -> VaultClient:
    """Factory para crear el cliente Vault según el entorno."""
    client = VaultClient(
        vault_addr=os.environ.get("VAULT_ADDR"),
        vault_role=os.environ.get("VAULT_ROLE"),
        secret_path=os.environ.get("VAULT_SECRET_PATH"),
        token=os.environ.get("VAULT_TOKEN"),
    )
    client.initialize()
    return client
EOF
```

### Salida Esperada

Archivos creados sin errores en `/home/student/msvc-labs/vault/app/`.

### Verificación

```bash
cd /home/student/msvc-labs/vault/app
python3 -c "import ast; ast.parse(open('vault_client.py').read()); print('vault_client.py: sintaxis OK')"
python3 -c "import ast; ast.parse(open('config.py').read()); print('config.py: sintaxis OK')"
```

---

## Paso 7: Refactorizar Microservicios para Usar Vault

**Objetivo:** Modificar el servicio de órdenes como ejemplo representativo, integrando el cliente Vault para obtener secretos en tiempo de ejecución con renovación automática.

### Instrucciones

1. Crear el archivo principal del servicio de órdenes refactorizado:

```bash
cat > /home/student/msvc-labs/vault/app/orders_main.py << 'EOF'
"""
Orders Service - Integrado con HashiCorp Vault.
Lee credenciales y configuración sensible desde Vault en tiempo de ejecución.
"""

import logging
from contextlib import asynccontextmanager

from fastapi import FastAPI, HTTPException, Depends
from pydantic import BaseModel

# Importar módulos Vault
import sys
sys.path.insert(0, "/app")
from vault_client import VaultClient
from config import ServiceConfig, create_vault_client

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

# Estado global del servicio
vault_client: VaultClient | None = None
service_config: ServiceConfig | None = None


@asynccontextmanager
async def lifespan(app: FastAPI):
    """Gestión del ciclo de vida: inicializa Vault al arrancar."""
    global vault_client, service_config

    logger.info("Iniciando Orders Service con integración Vault...")

    # Inicializar cliente Vault
    vault_client = create_vault_client()
    service_config = ServiceConfig.from_vault(vault_client)

    logger.info("Configuración cargada desde Vault:")
    logger.info("  database_url: %s", service_config.database_url[:30] + "...")
    logger.info("  api_key: %s...", service_config.api_key[:10])

    # Iniciar renovación automática de token (cada 30 min)
    await vault_client.start_token_renewal(interval=1800)

    yield

    # Cleanup
    if vault_client:
        await vault_client.stop_token_renewal()
        vault_client.close()
    logger.info("Orders Service detenido.")


app = FastAPI(
    title="Orders Service",
    version="2.0.0",
    description="Servicio de órdenes con gestión de secretos via Vault",
    lifespan=lifespan,
)


class OrderCreate(BaseModel):
    product_id: str
    quantity: int
    customer_id: str


class OrderResponse(BaseModel):
    order_id: str
    status: str
    product_id: str
    quantity: int


def get_config() -> ServiceConfig:
    """Dependencia para inyectar la configuración actual."""
    if service_config is None:
        raise HTTPException(
            status_code=503,
            detail="Servicio no inicializado: configuración Vault no disponible",
        )
    return service_config


@app.get("/health")
async def health_check():
    """Health check que verifica la conexión con Vault."""
    vault_ok = vault_client is not None and vault_client._client.is_authenticated()
    return {
        "status": "healthy" if vault_ok else "degraded",
        "vault_connected": vault_ok,
        "config_loaded": service_config is not None,
    }


@app.get("/config/status")
async def config_status(config: ServiceConfig = Depends(get_config)):
    """Muestra el estado de la configuración (sin exponer valores sensibles)."""
    return {
        "database_configured": bool(config.database_url),
        "api_key_configured": bool(config.api_key),
        "redis_configured": bool(config.redis_url),
        "extra_keys": list(config.extra.keys()),
    }


@app.post("/orders", response_model=OrderResponse)
async def create_order(
    order: OrderCreate,
    config: ServiceConfig = Depends(get_config),
):
    """Crea una orden (simulado). Demuestra uso de secretos inyectados."""
    import hashlib
    # Simular uso del api_key para firmar la orden
    signature = hashlib.sha256(
        f"{order.product_id}{config.api_key}".encode()
    ).hexdigest()[:8]

    return OrderResponse(
        order_id=f"ORD-{signature}",
        status="created",
        product_id=order.product_id,
        quantity=order.quantity,
    )


@app.post("/config/refresh")
async def refresh_config():
    """Endpoint para forzar refresco de configuración desde Vault."""
    if vault_client and service_config:
        service_config.refresh(vault_client)
        return {"status": "refreshed", "message": "Configuración actualizada desde Vault"}
    raise HTTPException(status_code=503, detail="Vault client no disponible")
EOF
```

2. Crear el Dockerfile para el servicio integrado con Vault:

```bash
cat > /home/student/msvc-labs/vault/app/Dockerfile << 'EOF'
FROM python:3.12-slim

WORKDIR /app

RUN pip install --no-cache-dir \
    fastapi==0.111.0 \
    uvicorn==0.29.0 \
    hvac==2.3.0 \
    pydantic==2.7.1

COPY vault_client.py config.py orders_main.py ./

EXPOSE 8001

CMD ["uvicorn", "orders_main:app", "--host", "0.0.0.0", "--port", "8001"]
EOF
```

3. Construir la imagen dentro de minikube:

```bash
eval $(minikube docker-env)
cd /home/student/msvc-labs/vault/app

docker build -t orders-service-vault:1.0 .
```

4. Crear el Deployment de Kubernetes para el servicio de órdenes:

```bash
cat > /home/student/msvc-labs/vault/k8s/deployments/orders-deployment.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: orders-service
  namespace: default
  labels:
    app: orders-service
    version: "2.0"
spec:
  replicas: 1
  selector:
    matchLabels:
      app: orders-service
  template:
    metadata:
      labels:
        app: orders-service
        version: "2.0"
    spec:
      serviceAccountName: orders-service-sa
      containers:
        - name: orders-service
          image: orders-service-vault:1.0
          imagePullPolicy: Never
          ports:
            - containerPort: 8001
          env:
            - name: VAULT_ADDR
              value: "http://vault.vault.svc.cluster.local:8200"
            - name: VAULT_ROLE
              value: "orders-role"
            - name: VAULT_SECRET_PATH
              value: "orders"
            # Token directo para modo dev (en producción usar K8s auth)
            - name: VAULT_TOKEN
              value: "root-token-msvc-lab"
          readinessProbe:
            httpGet:
              path: /health
              port: 8001
            initialDelaySeconds: 5
            periodSeconds: 10
          resources:
            requests:
              memory: "128Mi"
              cpu: "100m"
            limits:
              memory: "256Mi"
              cpu: "250m"
---
apiVersion: v1
kind: Service
metadata:
  name: orders-service
  namespace: default
spec:
  selector:
    app: orders-service
  ports:
    - port: 8001
      targetPort: 8001
  type: ClusterIP
EOF
```

5. Desplegar el servicio:

```bash
# Eliminar deployment anterior si existe
kubectl delete deployment orders-service --ignore-not-found=true
kubectl delete service orders-service --ignore-not-found=true

# Aplicar nuevo deployment
kubectl apply -f /home/student/msvc-labs/vault/k8s/deployments/orders-deployment.yaml

# Esperar a que esté listo
kubectl wait --for=condition=Available deployment/orders-service --timeout=60s
```

6. Verificar que el servicio lee secretos de Vault:

```bash
# Port-forward para acceder al servicio
kubectl port-forward svc/orders-service 8001:8001 &
PF_PID=$!
sleep 3

# Verificar health
curl -s http://localhost:8001/health | python3 -m json.tool

# Verificar estado de configuración
curl -s http://localhost:8001/config/status | python3 -m json.tool

# Crear una orden de prueba
curl -s -X POST http://localhost:8001/orders \
  -H "Content-Type: application/json" \
  -d '{"product_id": "PROD-001", "quantity": 5, "customer_id": "CUST-100"}' \
  | python3 -m json.tool

kill $PF_PID 2>/dev/null
```

### Salida Esperada

```json
{
    "status": "healthy",
    "vault_connected": true,
    "config_loaded": true
}
```

```json
{
    "database_configured": true,
    "api_key_configured": true,
    "redis_configured": true,
    "extra_keys": ["jwt_secret"]
}
```

```json
{
    "order_id": "ORD-a1b2c3d4",
    "status": "created",
    "product_id": "PROD-001",
    "quantity": 5
}
```

### Verificación

```bash
kubectl logs deployment/orders-service | grep -i "vault"
# Debe mostrar mensajes de inicialización exitosa con Vault
```

---

## Paso 8: Demostrar Rotación de Secretos Sin Reinicio

**Objetivo:** Actualizar secretos en Vault y verificar que el servicio adopta los nuevos valores sin necesidad de reiniciar el pod.

### Instrucciones

1. Verificar el valor actual del API key:

```bash
kubectl port-forward svc/orders-service 8001:8001 &
PF_PID=$!
sleep 2

# Crear orden con el api_key actual (genera un order_id basado en el key)
curl -s -X POST http://localhost:8001/orders \
  -H "Content-Type: application/json" \
  -d '{"product_id": "TEST-ROT", "quantity": 1, "customer_id": "C-1"}' \
  | python3 -m json.tool

echo "--- Order ID generado con api_key v1 ---"
```

2. Rotar el secreto en Vault:

```bash
# Actualizar el api_key a una nueva versión
vault kv put secret/orders \
  database_url="postgresql://orders_user:0rd3rs_s3cr3t@mslab-postgres:5432/orders_db" \
  api_key="orders-api-key-v2-ROTATED-xyz789" \
  redis_url="redis://mslab-redis:6379/0" \
  jwt_secret="orders-jwt-hmac-secret-2024"

echo "Secreto rotado en Vault. Nueva versión:"
vault kv get -field=api_key secret/orders
```

3. Forzar refresco de configuración en el servicio:

```bash
# Llamar al endpoint de refresco
curl -s -X POST http://localhost:8001/config/refresh | python3 -m json.tool
```

4. Verificar que el servicio usa el nuevo valor:

```bash
# Crear otra orden - el order_id será diferente porque el api_key cambió
curl -s -X POST http://localhost:8001/orders \
  -H "Content-Type: application/json" \
  -d '{"product_id": "TEST-ROT", "quantity": 1, "customer_id": "C-1"}' \
  | python3 -m json.tool

echo "--- Order ID generado con api_key v2 (debe ser diferente) ---"

kill $PF_PID 2>/dev/null
```

5. Verificar que el pod NO se reinició:

```bash
kubectl get pods -l app=orders-service -o wide
# La columna RESTARTS debe ser 0
# La columna AGE debe ser la misma que antes de la rotación
```

6. Verificar el historial de versiones en Vault:

```bash
vault kv metadata get secret/orders
# Debe mostrar current_version: 2 (o más si se ejecutó más de una vez)
```

### Salida Esperada

```
--- Antes de la rotación ---
{
    "order_id": "ORD-a1b2c3d4",
    ...
}

--- Después de la rotación ---
{
    "order_id": "ORD-e5f6g7h8",  <-- ID diferente confirma nuevo api_key
    ...
}
```

```
RESTARTS: 0
```

### Verificación

```bash
# Los logs del servicio deben mostrar el refresco
kubectl logs deployment/orders-service | grep -i "refrescad"
```

---

## Paso 9: Auditar Accesos a Secretos

**Objetivo:** Analizar el audit log de Vault para demostrar trazabilidad completa de quién accedió a qué secretos y cuándo.

### Instrucciones

1. Generar actividad auditable realizando varias operaciones:

```bash
# Leer secretos de diferentes servicios
vault kv get secret/orders > /dev/null
vault kv get secret/inventory > /dev/null
vault kv get secret/users > /dev/null

# Intentar acceso no autorizado (con token restringido)
ORDERS_TOKEN=$(vault token create -policy=orders-policy -format=json | jq -r '.auth.client_token')
VAULT_TOKEN=$ORDERS_TOKEN vault kv get secret/inventory 2>/dev/null || true
```

2. Extraer y analizar las entradas de auditoría:

```bash
# Obtener logs del pod de Vault (donde se escribe el audit log a stdout)
kubectl logs -n vault vault-0 --tail=50 > /tmp/vault-audit-raw.log

# Filtrar y formatear entradas relevantes
cat /tmp/vault-audit-raw.log | \
  grep '"type":"response"' | \
  python3 -c "
import sys, json

print('=' * 80)
print(f'{\"TIMESTAMP\":<28} {\"OPERACIÓN\":<10} {\"RUTA\":<30} {\"RESULTADO\"}')
print('=' * 80)

for line in sys.stdin:
    line = line.strip()
    if not line:
        continue
    try:
        entry = json.loads(line)
        if entry.get('type') == 'response':
            ts = entry.get('time', 'N/A')[:19]
            req = entry.get('request', {})
            path = req.get('path', 'N/A')
            operation = req.get('operation', 'N/A')
            resp = entry.get('response', {})
            # Detectar errores (permission denied)
            error = entry.get('error', '')
            status = 'DENIED' if error else 'OK'
            if 'secret/' in path:
                print(f'{ts:<28} {operation:<10} {path:<30} {status}')
    except json.JSONDecodeError:
        continue
" 2>/dev/null || echo "Nota: el formato de audit log puede variar en modo dev"
```

3. Crear un script de auditoría reutilizable:

```bash
cat > /home/student/msvc-labs/vault/scripts/audit-report.sh << 'EOF'
#!/bin/bash
# Script para generar reporte de auditoría de accesos a Vault
echo "=========================================="
echo " REPORTE DE AUDITORÍA - HashiCorp Vault"
echo " Generado: $(date)"
echo "=========================================="
echo ""

echo "--- Versiones actuales de secretos ---"
for svc in orders inventory users; do
  version=$(vault kv metadata get -format=json secret/$svc 2>/dev/null | jq -r '.data.current_version')
  echo "  secret/$svc: versión $version"
done

echo ""
echo "--- Políticas activas ---"
vault policy list 2>/dev/null | grep -v "^root$\|^default$"

echo ""
echo "--- Tokens activos (últimas 5 entradas del audit log) ---"
kubectl logs -n vault vault-0 --tail=100 2>/dev/null | \
  grep '"type":"request"' | \
  grep "secret/data" | \
  tail -5 | \
  python3 -c "
import sys, json
for line in sys.stdin:
    try:
        e = json.loads(line.strip())
        req = e.get('request', {})
        print(f\"  [{e.get('time','')[:19]}] {req.get('operation','')}: {req.get('path','')}\")
    except: pass
" 2>/dev/null

echo ""
echo "=========================================="
EOF

chmod +x /home/student/msvc-labs/vault/scripts/audit-report.sh
```

4. Ejecutar el reporte:

```bash
/home/student/msvc-labs/vault/scripts/audit-report.sh
```

### Salida Esperada

```
==========================================
 REPORTE DE AUDITORÍA - HashiCorp Vault
 Generado: Mon Jul 15 14:30:00 UTC 2024
==========================================

--- Versiones actuales de secretos ---
  secret/orders: versión 2
  secret/inventory: versión 1
  secret/users: versión 1

--- Políticas activas ---
inventory-policy
orders-policy
users-policy

--- Tokens activos (últimas 5 entradas del audit log) ---
  [2024-07-15T14:28:] read: secret/data/orders
  [2024-07-15T14:29:] read: secret/data/inventory
  [2024-07-15T14:29:] read: secret/data/users
  [2024-07-15T14:30:] read: secret/data/inventory

==========================================
```

### Verificación

```bash
# Confirmar que los accesos denegados se registran
kubectl logs -n vault vault-0 | grep -c "permission denied" || echo "0 denegaciones (normal si se usó root token)"
```

---

## Validación y Pruebas

Ejecutar la siguiente batería de validaciones para confirmar que todo el laboratorio funciona correctamente:

```bash
#!/bin/bash
echo "============================================"
echo " VALIDACIÓN COMPLETA - Lab 07-00-01"
echo "============================================"
PASS=0
FAIL=0

# Test 1: Vault está corriendo
echo -n "[1/7] Vault pod running... "
if kubectl get pod vault-0 -n vault -o jsonpath='{.status.phase}' | grep -q "Running"; then
  echo "PASS"; ((PASS++))
else
  echo "FAIL"; ((FAIL++))
fi

# Test 2: Motor KV v2 tiene secretos
echo -n "[2/7] Secretos en KV v2... "
if vault kv get -format=json secret/orders > /dev/null 2>&1; then
  echo "PASS"; ((PASS++))
else
  echo "FAIL"; ((FAIL++))
fi

# Test 3: Políticas cargadas
echo -n "[3/7] Políticas de acceso... "
POLICIES=$(vault policy list 2>/dev/null | grep -c "policy")
if [ "$POLICIES" -ge 3 ]; then
  echo "PASS ($POLICIES políticas)"; ((PASS++))
else
  echo "FAIL ($POLICIES políticas)"; ((FAIL++))
fi

# Test 4: Kubernetes auth habilitado
echo -n "[4/7] Kubernetes auth method... "
if vault auth list 2>/dev/null | grep -q "kubernetes"; then
  echo "PASS"; ((PASS++))
else
  echo "FAIL"; ((FAIL++))
fi

# Test 5: ServiceAccounts creados
echo -n "[5/7] ServiceAccounts... "
SA_COUNT=$(kubectl get sa -o name | grep -c "service-sa")
if [ "$SA_COUNT" -ge 3 ]; then
  echo "PASS ($SA_COUNT SAs)"; ((PASS++))
else
  echo "FAIL ($SA_COUNT SAs)"; ((FAIL++))
fi

# Test 6: Orders service con Vault
echo -n "[6/7] Orders service health... "
kubectl port-forward svc/orders-service 8001:8001 > /dev/null 2>&1 &
PF=$!
sleep 3
HEALTH=$(curl -s http://localhost:8001/health 2>/dev/null | python3 -c "import sys,json; print(json.load(sys.stdin).get('vault_connected',''))" 2>/dev/null)
kill $PF 2>/dev/null
if [ "$HEALTH" = "True" ]; then
  echo "PASS"; ((PASS++))
else
  echo "FAIL (vault_connected=$HEALTH)"; ((FAIL++))
fi

# Test 7: Aislamiento de políticas
echo -n "[7/7] Aislamiento de secretos... "
TEST_TOKEN=$(vault token create -policy=orders-policy -format=json 2>/dev/null | jq -r '.auth.client_token')
DENIED=$(VAULT_TOKEN=$TEST_TOKEN vault kv get secret/inventory 2>&1 | grep -c "permission denied")
if [ "$DENIED" -ge 1 ]; then
  echo "PASS (acceso denegado correctamente)"; ((PASS++))
else
  echo "FAIL"; ((FAIL++))
fi

echo ""
echo "============================================"
echo " Resultado: $PASS/7 pruebas exitosas"
echo "============================================"
if [ "$FAIL" -eq 0 ]; then
  echo " ✅ LABORATORIO COMPLETADO EXITOSAMENTE"
else
  echo " ⚠️  $FAIL prueba(s) fallida(s) - revisar pasos"
fi
```

---

## Solución de Problemas

### Problema 1: El pod de Vault no arranca o queda en CrashLoopBackOff

**Síntomas:**
```
$ kubectl get pods -n vault
NAME      READY   STATUS             RESTARTS   AGE
vault-0   0/1     CrashLoopBackOff   3          2m
```

**Causa:** Recursos insuficientes en minikube o conflicto de puertos con el NodePort 30820. En algunos casos, el PersistentVolumeClaim para audit storage no puede aprovisionarse.

**Solución:**
```bash
# Verificar logs del pod
kubectl logs -n vault vault-0 --previous

# Si es problema de recursos, aumentar los de minikube
minikube stop
minikube start --cpus=4 --memory=8192

# Si es problema de PVC, deshabilitar auditStorage en values.yaml
# Editar vault-values.yaml: auditStorage.enabled: false
# Y reinstalar:
helm uninstall vault -n vault
helm install vault hashicorp/vault \
  --namespace vault \
  --values /home/student/msvc-labs/vault/helm/vault-values.yaml \
  --version 0.28.0

# Si es conflicto de puerto, cambiar nodePort a 30821 en vault-values.yaml
```

### Problema 2: El servicio FastAPI no puede conectarse a Vault (ConnectionError o timeout)

**Síntomas:**
```
kubectl logs deployment/orders-service
...
hvac.exceptions.VaultError: Error en autenticación Kubernetes: ...
ConnectionError: HTTPConnectionPool(host='vault.vault.svc.cluster.local', port=8200): Max retries exceeded
```

**Causa:** El DNS interno de Kubernetes no resuelve `vault.vault.svc.cluster.local` porque el servicio de Vault no está expuesto correctamente, o la variable `VAULT_ADDR` tiene un valor incorrecto.

**Solución:**
```bash
# Verificar que el servicio Vault existe y tiene endpoints
kubectl get svc -n vault
kubectl get endpoints vault -n vault

# Probar conectividad desde un pod temporal
kubectl run test-dns --rm -it --image=busybox --restart=Never -- \
  wget -qO- http://vault.vault.svc.cluster.local:8200/v1/sys/health

# Si DNS no funciona, usar la IP del servicio directamente
VAULT_SVC_IP=$(kubectl get svc vault -n vault -o jsonpath='{.spec.clusterIP}')
echo "Usar VAULT_ADDR=http://${VAULT_SVC_IP}:8200"

# Actualizar el deployment con la IP correcta
kubectl set env deployment/orders-service \
  VAULT_ADDR="http://${VAULT_SVC_IP}:8200"

# Reiniciar el deployment
kubectl rollout restart deployment/orders-service
kubectl rollout status deployment/orders-service
```

---

## Limpieza

```bash
# Eliminar deployments de servicios con Vault
kubectl delete -f /home/student/msvc-labs/vault/k8s/deployments/ --ignore-not-found=true
kubectl delete -f /home/student/msvc-labs/vault/k8s/serviceaccounts.yaml --ignore-not-found=true

# Desinstalar Vault
helm uninstall vault -n vault
kubectl delete namespace vault

# Eliminar imágenes Docker de minikube
eval $(minikube docker-env)
docker rmi orders-service-vault:1.0 2>/dev/null

# Limpiar port-forwards huérfanos
pkill -f "port-forward.*8001" 2>/dev/null

# Opcional: eliminar archivos del laboratorio
# rm -rf /home/student/msvc-labs/vault/

echo "Limpieza completada."
```

---

## Resumen

En este laboratorio se implementó una integración completa entre HashiCorp Vault y microservicios FastAPI que demuestra:

| Concepto | Implementación |
|----------|---------------|
| **Gestión centralizada** | Motor KV v2 con rutas separadas por servicio |
| **Acceso mínimo** | Políticas HCL con `deny` explícito entre servicios |
| **Autenticación nativa** | Kubernetes auth method con ServiceAccounts |
| **Resiliencia** | Caché local con fallback a valores stale |
| **Rotación sin downtime** | Endpoint de refresco + background task de renovación |
| **Auditoría** | Audit log con trazabilidad de operaciones y accesos denegados |

### Principios Clave Aplicados

1. **Separación de secretos del código:** Los secretos nunca se embeben en imágenes Docker ni en repositorios Git
2. **Principio de menor privilegio:** Cada servicio accede solo a su ruta de secretos
3. **Configuración como servidor centralizado:** Patrón 3 de distribución de configuración (consulta activa a servidor)
4. **Resiliencia ante fallos:** Caché stale evita interrupciones si Vault no está disponible temporalmente

### Recursos Adicionales

- [Vault Kubernetes Auth Method](https://developer.hashicorp.com/vault/docs/auth/kubernetes)
- [hvac Python Client Documentation](https://hvac.readthedocs.io/en/stable/)
- [Vault KV v2 Secrets Engine](https://developer.hashicorp.com/vault/docs/secrets/kv/kv-v2)
- [Vault Audit Devices](https://developer.hashicorp.com/vault/docs/audit)
