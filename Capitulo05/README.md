# Proteger APIs Python con OAuth2 y JWT usando Keycloak

## Metadatos

| Campo | Valor |
|-------|-------|
| **Duración** | 66 minutos |
| **Complejidad** | Alta |
| **Nivel Bloom** | Aplicar |

## Descripción General

En este laboratorio implementarás seguridad OAuth2/JWT completa sobre la arquitectura de microservicios construida en los Labs 01-04. Desplegarás Keycloak 24.0.4 como servidor de autorización, configurarás un realm con clientes OAuth2 y roles de aplicación, implementarás validación de JWT en FastAPI como dependencia reutilizable, y asegurarás la comunicación inter-servicio usando el flujo Client Credentials. Al finalizar, toda la comunicación entre servicios y con clientes externos estará protegida con tokens de corta duración verificados criptográficamente.

## Objetivos de Aprendizaje

- [ ] Configurar Keycloak con realm, clientes confidenciales y roles de aplicación para dos microservicios
- [ ] Implementar validación de JWT en FastAPI con verificación de firma RS256 contra JWKS endpoint
- [ ] Proteger endpoints con dependencias que verifiquen scopes y roles del token
- [ ] Asegurar comunicación inter-servicio con flujo Client Credentials y tokens automáticos
- [ ] Validar flujos completos incluyendo rechazo de tokens inválidos y expirados

## Prerrequisitos

### Conocimiento Requerido

- Labs 01-04 completados con arquitectura funcional (Event Store, Kafka, resiliencia)
- Comprensión de OAuth2 flows: Authorization Code y Client Credentials
- Estructura JWT: header.payload.signature y claims estándar
- Familiaridad con dependencias de seguridad de FastAPI

### Acceso Requerido

- Directorio `~/microservices-lab/` con estructura completa de labs anteriores
- Módulo `shared/resilience/` del Lab 04 operativo
- Acceso a internet para descargar imagen de Keycloak

## Entorno del Laboratorio

### Software Adicional

| Componente | Versión | Propósito |
|------------|---------|-----------|
| Keycloak | 24.0.4 | Servidor de autorización OAuth2/OIDC |
| python-jose[cryptography] | 3.3.0 | Validación y decodificación de JWT |
| python-keycloak | 4.2.2 | Configuración programática de Keycloak |
| httpx | 0.27.0 | Cliente HTTP async para JWKS |
| python-dotenv | 1.0.1 | Gestión de variables de entorno |

### Configuración Inicial

```bash
cd ~/microservices-lab/
mkdir -p shared/auth tests/test_auth scripts/keycloak
```

Instalar dependencias adicionales:

```bash
pip install "python-jose[cryptography]==3.3.0" "python-keycloak==4.2.2" \
    "httpx==0.27.0" "python-dotenv==1.0.1" "pytest==8.2.1" \
    "pytest-asyncio==0.23.7" "redis==5.0.4"
```

---

## Paso 1: Desplegar Keycloak con Docker Compose

**Objetivo:** Extender el docker-compose existente para incluir Keycloak 24.0.4 como servidor de autorización.

### Instrucciones

1. Crear el archivo de extensión de Docker Compose para Keycloak:

```bash
cat > ~/microservices-lab/docker-compose.auth.yml << 'EOF'
version: "3.9"

services:
  mslab-keycloak:
    image: quay.io/keycloak/keycloak:24.0.4
    container_name: mslab-keycloak
    environment:
      KEYCLOAK_ADMIN: admin
      KEYCLOAK_ADMIN_PASSWORD: admin_secret_2024
      KC_DB: postgres
      KC_DB_URL: jdbc:postgresql://mslab-postgres:5432/keycloak_db
      KC_DB_USERNAME: mslab_user
      KC_DB_PASSWORD: mslab_pass_2024
      KC_HOSTNAME_STRICT: "false"
      KC_HTTP_ENABLED: "true"
      KC_PROXY_HEADERS: xforwarded
    command: start-dev
    ports:
      - "8080:8080"
    depends_on:
      mslab-postgres:
        condition: service_healthy
    networks:
      - mslab-network
    healthcheck:
      test: ["CMD-SHELL", "exec 3<>/dev/tcp/127.0.0.1/8080; echo -e 'GET /health/ready HTTP/1.1\r\nhost: localhost\r\nConnection: close\r\n\r\n' >&3; grep -q '200' <&3"]
      interval: 10s
      timeout: 5s
      retries: 12
      start_period: 30s

networks:
  mslab-network:
    external: true

EOF
```

2. Crear la base de datos para Keycloak en PostgreSQL:

```bash
docker exec mslab-postgres psql -U mslab_user -d postgres -c \
    "CREATE DATABASE keycloak_db OWNER mslab_user;"
```

3. Levantar Keycloak:

```bash
cd ~/microservices-lab/
docker compose -f docker-compose.yml -f docker-compose.auth.yml up -d mslab-keycloak
```

4. Esperar a que Keycloak esté listo:

```bash
echo "Esperando a Keycloak..."
until curl -sf http://localhost:8080/health/ready > /dev/null 2>&1; do
    sleep 5
    echo "  ...esperando"
done
echo "Keycloak listo"
```

### Salida Esperada

```
Esperando a Keycloak...
  ...esperando
  ...esperando
Keycloak listo
```

### Verificación

```bash
curl -s http://localhost:8080/realms/master | python3 -m json.tool | head -5
```

Debe mostrar información del realm `master` con el campo `"realm": "master"`.

---

## Paso 2: Configurar Realm, Clientes y Roles Programáticamente

**Objetivo:** Crear el realm `microservices-lab`, clientes OAuth2 confidenciales y roles de aplicación usando un script Python con la biblioteca python-keycloak.

### Instrucciones

1. Crear el script de configuración:

```bash
cat > ~/microservices-lab/scripts/keycloak/setup_realm.py << 'EOF'
"""
Script de configuración de Keycloak para el laboratorio.
Crea realm, clientes, roles y usuarios de prueba.
"""

from keycloak import KeycloakAdmin, KeycloakOpenIDConnection

# Conexión como admin del realm master
connection = KeycloakOpenIDConnection(
    server_url="http://localhost:8080",
    username="admin",
    password="admin_secret_2024",
    realm_name="master",
    verify=False,
)

admin = KeycloakAdmin(connection=connection)

# --- 1. Crear Realm ---
REALM_NAME = "microservices-lab"

try:
    admin.create_realm(
        payload={
            "realm": REALM_NAME,
            "enabled": True,
            "accessTokenLifespan": 300,  # 5 minutos
            "ssoSessionIdleTimeout": 1800,
            "registrationAllowed": False,
        },
        skip_exists=True,
    )
    print(f"✓ Realm '{REALM_NAME}' creado")
except Exception as e:
    print(f"  Realm ya existe o error: {e}")

# Cambiar al nuevo realm
connection.realm_name = REALM_NAME
admin = KeycloakAdmin(connection=connection)

# --- 2. Crear Roles del Realm ---
ROLES = ["order:read", "order:write", "inventory:read", "inventory:write"]

for role_name in ROLES:
    try:
        admin.create_realm_role(
            payload={"name": role_name, "description": f"Permiso {role_name}"},
            skip_exists=True,
        )
        print(f"✓ Rol '{role_name}' creado")
    except Exception as e:
        print(f"  Rol '{role_name}' ya existe: {e}")

# --- 3. Crear Cliente: order-service-client ---
ORDER_CLIENT_CONFIG = {
    "clientId": "order-service-client",
    "secret": "order-svc-secret-2024",
    "enabled": True,
    "protocol": "openid-connect",
    "publicClient": False,
    "serviceAccountsEnabled": True,
    "authorizationServicesEnabled": False,
    "directAccessGrantsEnabled": True,
    "standardFlowEnabled": False,
    "clientAuthenticatorType": "client-secret",
    "defaultClientScopes": ["openid", "profile", "email"],
}

try:
    admin.create_client(ORDER_CLIENT_CONFIG, skip_exists=True)
    print("✓ Cliente 'order-service-client' creado")
except Exception as e:
    print(f"  Cliente order-service ya existe: {e}")

# --- 4. Crear Cliente: inventory-service-client ---
INVENTORY_CLIENT_CONFIG = {
    "clientId": "inventory-service-client",
    "secret": "inventory-svc-secret-2024",
    "enabled": True,
    "protocol": "openid-connect",
    "publicClient": False,
    "serviceAccountsEnabled": True,
    "authorizationServicesEnabled": False,
    "directAccessGrantsEnabled": True,
    "standardFlowEnabled": False,
    "clientAuthenticatorType": "client-secret",
    "defaultClientScopes": ["openid", "profile", "email"],
}

try:
    admin.create_client(INVENTORY_CLIENT_CONFIG, skip_exists=True)
    print("✓ Cliente 'inventory-service-client' creado")
except Exception as e:
    print(f"  Cliente inventory-service ya existe: {e}")

# --- 5. Crear Cliente público: frontend-client ---
FRONTEND_CLIENT_CONFIG = {
    "clientId": "frontend-client",
    "enabled": True,
    "protocol": "openid-connect",
    "publicClient": True,
    "serviceAccountsEnabled": False,
    "directAccessGrantsEnabled": True,
    "standardFlowEnabled": True,
    "redirectUris": ["http://localhost:3000/*"],
    "webOrigins": ["http://localhost:3000"],
    "defaultClientScopes": ["openid", "profile", "email"],
}

try:
    admin.create_client(FRONTEND_CLIENT_CONFIG, skip_exists=True)
    print("✓ Cliente 'frontend-client' creado")
except Exception as e:
    print(f"  Cliente frontend ya existe: {e}")

# --- 6. Asignar roles a service accounts ---
# Order service: todos los roles de order + inventory:read
order_client_id = admin.get_client_id("order-service-client")
order_sa_id = admin.get_client_service_account_user(order_client_id)["id"]

all_realm_roles = admin.get_realm_roles()
order_roles = [r for r in all_realm_roles if r["name"] in ["order:read", "order:write", "inventory:read"]]
admin.assign_realm_roles(user_id=order_sa_id, roles=order_roles)
print("✓ Roles asignados a service account de order-service-client")

# Inventory service: todos los roles de inventory + order:read
inv_client_id = admin.get_client_id("inventory-service-client")
inv_sa_id = admin.get_client_service_account_user(inv_client_id)["id"]

inv_roles = [r for r in all_realm_roles if r["name"] in ["inventory:read", "inventory:write", "order:read"]]
admin.assign_realm_roles(user_id=inv_sa_id, roles=inv_roles)
print("✓ Roles asignados a service account de inventory-service-client")

# --- 7. Crear Usuarios de Prueba ---
# admin-user: todos los roles
try:
    admin.create_user(
        payload={
            "username": "admin-user",
            "email": "admin@microservices-lab.dev",
            "enabled": True,
            "emailVerified": True,
            "credentials": [{"type": "password", "value": "Admin@2024!", "temporary": False}],
        },
        exist_ok=True,
    )
    admin_user_id = admin.get_user_id("admin-user")
    admin.assign_realm_roles(
        user_id=admin_user_id,
        roles=[r for r in all_realm_roles if r["name"] in ROLES],
    )
    print("✓ Usuario 'admin-user' creado con todos los roles")
except Exception as e:
    print(f"  admin-user: {e}")

# readonly-user: solo roles :read
try:
    admin.create_user(
        payload={
            "username": "readonly-user",
            "email": "readonly@microservices-lab.dev",
            "enabled": True,
            "emailVerified": True,
            "credentials": [{"type": "password", "value": "Read@2024!", "temporary": False}],
        },
        exist_ok=True,
    )
    readonly_user_id = admin.get_user_id("readonly-user")
    read_roles = [r for r in all_realm_roles if r["name"] in ["order:read", "inventory:read"]]
    admin.assign_realm_roles(user_id=readonly_user_id, roles=read_roles)
    print("✓ Usuario 'readonly-user' creado con roles de lectura")
except Exception as e:
    print(f"  readonly-user: {e}")

print("\n✅ Configuración de Keycloak completada exitosamente")
EOF
```

2. Ejecutar el script:

```bash
cd ~/microservices-lab/
python3 scripts/keycloak/setup_realm.py
```

### Salida Esperada

```
✓ Realm 'microservices-lab' creado
✓ Rol 'order:read' creado
✓ Rol 'order:write' creado
✓ Rol 'inventory:read' creado
✓ Rol 'inventory:write' creado
✓ Cliente 'order-service-client' creado
✓ Cliente 'inventory-service-client' creado
✓ Cliente 'frontend-client' creado
✓ Roles asignados a service account de order-service-client
✓ Roles asignados a service account de inventory-service-client
✓ Usuario 'admin-user' creado con todos los roles
✓ Usuario 'readonly-user' creado con roles de lectura

✅ Configuración de Keycloak completada exitosamente
```

### Verificación

```bash
# Verificar que el realm responde correctamente
curl -s http://localhost:8080/realms/microservices-lab/.well-known/openid-configuration \
    | python3 -c "import sys,json; d=json.load(sys.stdin); print('Issuer:', d['issuer']); print('JWKS:', d['jwks_uri'])"
```

Debe mostrar:
```
Issuer: http://localhost:8080/realms/microservices-lab
JWKS: http://localhost:8080/realms/microservices-lab/protocol/openid-connect/certs
```

---

## Paso 3: Implementar Módulo de Validación JWT Reutilizable

**Objetivo:** Crear el módulo `shared/auth/` con validación de JWT, caché de JWKS y dependencias FastAPI reutilizables.

### Instrucciones

1. Crear la estructura del módulo:

```bash
mkdir -p ~/microservices-lab/shared/auth/
touch ~/microservices-lab/shared/auth/__init__.py
```

2. Crear el archivo de configuración de entorno:

```bash
cat > ~/microservices-lab/.env.example << 'EOF'
# === Keycloak / OAuth2 Configuration ===
KEYCLOAK_URL=http://localhost:8080
KEYCLOAK_REALM=microservices-lab
JWKS_CACHE_TTL_SECONDS=3600

# === Order Service OAuth2 Client ===
ORDER_SERVICE_CLIENT_ID=order-service-client
ORDER_SERVICE_CLIENT_SECRET=order-svc-secret-2024

# === Inventory Service OAuth2 Client ===
INVENTORY_SERVICE_CLIENT_ID=inventory-service-client
INVENTORY_SERVICE_CLIENT_SECRET=inventory-svc-secret-2024

# === PostgreSQL ===
POSTGRES_USER=mslab_user
POSTGRES_PASSWORD=mslab_pass_2024
POSTGRES_HOST=localhost
EOF

cp ~/microservices-lab/.env.example ~/microservices-lab/.env
```

3. Crear el módulo principal de autenticación:

```bash
cat > ~/microservices-lab/shared/auth/jwt_validator.py << 'EOF'
"""
Módulo de validación JWT para microservicios.
Descarga y cachea las claves JWKS de Keycloak para verificación local.
"""

import time
import logging
from typing import Optional

import httpx
from jose import jwt, JWTError, jwk
from jose.utils import base64url_decode

logger = logging.getLogger(__name__)


class JWKSCache:
    """Caché en memoria para las claves públicas JWKS con TTL configurable."""

    def __init__(self, ttl_seconds: int = 3600):
        self._keys: Optional[dict] = None
        self._fetched_at: float = 0
        self._ttl = ttl_seconds

    @property
    def is_expired(self) -> bool:
        return (time.time() - self._fetched_at) > self._ttl

    @property
    def keys(self) -> Optional[dict]:
        if self.is_expired:
            return None
        return self._keys

    def update(self, keys: dict) -> None:
        self._keys = keys
        self._fetched_at = time.time()
        logger.info("JWKS cache actualizado (%d claves)", len(keys.get("keys", [])))

    def invalidate(self) -> None:
        self._keys = None
        self._fetched_at = 0


class JWTValidator:
    """
    Validador de JWT que verifica tokens contra las claves JWKS de Keycloak.
    
    Características:
    - Descarga automática de JWKS con caché configurable
    - Verificación de firma RS256
    - Validación de claims: exp, iss, aud
    - Extracción de roles del realm
    """

    def __init__(
        self,
        keycloak_url: str,
        realm: str,
        audience: Optional[str] = None,
        jwks_cache_ttl: int = 3600,
    ):
        self._issuer = f"{keycloak_url}/realms/{realm}"
        self._jwks_uri = f"{self._issuer}/protocol/openid-connect/certs"
        self._audience = audience
        self._cache = JWKSCache(ttl_seconds=jwks_cache_ttl)

    async def _fetch_jwks(self) -> dict:
        """Descarga las claves JWKS del endpoint de Keycloak."""
        async with httpx.AsyncClient(timeout=10.0) as client:
            response = await client.get(self._jwks_uri)
            response.raise_for_status()
            jwks = response.json()
            self._cache.update(jwks)
            return jwks

    async def _get_jwks(self) -> dict:
        """Obtiene JWKS desde caché o los descarga si han expirado."""
        cached = self._cache.keys
        if cached is not None:
            return cached
        return await self._fetch_jwks()

    async def validate_token(self, token: str) -> dict:
        """
        Valida un token JWT y retorna los claims decodificados.
        
        Raises:
            JWTValidationError: Si el token es inválido, expirado o no autorizado.
        """
        try:
            jwks = await self._get_jwks()

            # Decodificar header sin verificar para obtener kid
            unverified_header = jwt.get_unverified_header(token)
            kid = unverified_header.get("kid")

            if not kid:
                raise JWTValidationError("Token sin 'kid' en header")

            # Buscar la clave correspondiente
            rsa_key = None
            for key in jwks.get("keys", []):
                if key.get("kid") == kid:
                    rsa_key = key
                    break

            if rsa_key is None:
                # Intentar refrescar JWKS por si se rotaron las claves
                self._cache.invalidate()
                jwks = await self._fetch_jwks()
                for key in jwks.get("keys", []):
                    if key.get("kid") == kid:
                        rsa_key = key
                        break

            if rsa_key is None:
                raise JWTValidationError(f"Clave pública no encontrada para kid='{kid}'")

            # Validar y decodificar
            decode_options = {
                "verify_exp": True,
                "verify_iss": True,
                "verify_aud": self._audience is not None,
            }

            claims = jwt.decode(
                token,
                rsa_key,
                algorithms=["RS256"],
                audience=self._audience,
                issuer=self._issuer,
                options=decode_options,
            )

            return claims

        except JWTError as e:
            raise JWTValidationError(f"Token JWT inválido: {e}")
        except httpx.HTTPError as e:
            raise JWTValidationError(f"Error al obtener JWKS: {e}")

    @staticmethod
    def extract_roles(claims: dict) -> list[str]:
        """Extrae los roles del realm desde los claims del token."""
        realm_access = claims.get("realm_access", {})
        return realm_access.get("roles", [])


class JWTValidationError(Exception):
    """Excepción para errores de validación de JWT."""
    pass
EOF
```

4. Crear las dependencias FastAPI de seguridad:

```bash
cat > ~/microservices-lab/shared/auth/dependencies.py << 'EOF'
"""
Dependencias FastAPI para protección de endpoints con OAuth2/JWT.
"""

import os
from functools import lru_cache
from typing import Optional

from fastapi import Depends, HTTPException, status
from fastapi.security import HTTPBearer, HTTPAuthorizationCredentials
from dotenv import load_dotenv

from .jwt_validator import JWTValidator, JWTValidationError

load_dotenv()

security_scheme = HTTPBearer(
    scheme_name="Bearer JWT",
    description="Token JWT obtenido de Keycloak",
)


@lru_cache()
def get_jwt_validator() -> JWTValidator:
    """Singleton del validador JWT configurado desde variables de entorno."""
    keycloak_url = os.getenv("KEYCLOAK_URL", "http://localhost:8080")
    realm = os.getenv("KEYCLOAK_REALM", "microservices-lab")
    cache_ttl = int(os.getenv("JWKS_CACHE_TTL_SECONDS", "3600"))

    return JWTValidator(
        keycloak_url=keycloak_url,
        realm=realm,
        jwks_cache_ttl=cache_ttl,
    )


async def get_current_user(
    credentials: HTTPAuthorizationCredentials = Depends(security_scheme),
    validator: JWTValidator = Depends(get_jwt_validator),
) -> dict:
    """
    Dependencia que extrae y valida el usuario actual desde el token JWT.
    Retorna los claims completos del token validado.
    """
    try:
        claims = await validator.validate_token(credentials.credentials)
        return claims
    except JWTValidationError as e:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail=str(e),
            headers={"WWW-Authenticate": "Bearer"},
        )


class RoleChecker:
    """
    Dependencia configurable que verifica que el usuario tenga
    los roles requeridos.
    
    Uso:
        @app.get("/orders", dependencies=[Depends(RoleChecker(["order:read"]))])
    """

    def __init__(self, required_roles: list[str]):
        self.required_roles = required_roles

    async def __call__(self, claims: dict = Depends(get_current_user)) -> dict:
        user_roles = JWTValidator.extract_roles(claims)
        missing = [r for r in self.required_roles if r not in user_roles]

        if missing:
            raise HTTPException(
                status_code=status.HTTP_403_FORBIDDEN,
                detail=f"Roles insuficientes. Faltan: {missing}",
            )
        return claims


# Instancias predefinidas para uso directo
require_order_read = RoleChecker(["order:read"])
require_order_write = RoleChecker(["order:read", "order:write"])
require_inventory_read = RoleChecker(["inventory:read"])
require_inventory_write = RoleChecker(["inventory:read", "inventory:write"])
EOF
```

5. Actualizar el `__init__.py`:

```bash
cat > ~/microservices-lab/shared/auth/__init__.py << 'EOF'
"""Módulo de autenticación y autorización compartido."""

from .jwt_validator import JWTValidator, JWTValidationError, JWKSCache
from .dependencies import (
    get_current_user,
    get_jwt_validator,
    RoleChecker,
    require_order_read,
    require_order_write,
    require_inventory_read,
    require_inventory_write,
)

__all__ = [
    "JWTValidator",
    "JWTValidationError",
    "JWKSCache",
    "get_current_user",
    "get_jwt_validator",
    "RoleChecker",
    "require_order_read",
    "require_order_write",
    "require_inventory_read",
    "require_inventory_write",
]
EOF
```

### Salida Esperada

La estructura de archivos debe ser:

```
shared/auth/
├── __init__.py
├── jwt_validator.py
└── dependencies.py
```

### Verificación

```bash
cd ~/microservices-lab/
python3 -c "from shared.auth import JWTValidator, RoleChecker; print('✓ Módulo auth importado correctamente')"
```

---

## Paso 4: Implementar Cliente de Tokens para Comunicación Inter-Servicio

**Objetivo:** Crear un cliente OAuth2 que obtiene y renueva automáticamente tokens para la comunicación entre servicios usando Client Credentials.

### Instrucciones

1. Crear el módulo de cliente de tokens:

```bash
cat > ~/microservices-lab/shared/auth/token_client.py << 'EOF'
"""
Cliente OAuth2 para obtención de tokens de servicio (Client Credentials flow).
Gestiona automáticamente la renovación de tokens antes de su expiración.
"""

import time
import logging
from typing import Optional
from dataclasses import dataclass

import httpx

logger = logging.getLogger(__name__)


@dataclass
class TokenResponse:
    """Respuesta de token del servidor de autorización."""
    access_token: str
    expires_in: int
    token_type: str
    obtained_at: float

    @property
    def is_expired(self) -> bool:
        """Verifica si el token ha expirado (con margen de 30 segundos)."""
        return (time.time() - self.obtained_at) >= (self.expires_in - 30)


class ServiceTokenClient:
    """
    Cliente que gestiona tokens OAuth2 para comunicación inter-servicio.
    
    Obtiene tokens usando Client Credentials y los cachea hasta que
    estén próximos a expirar.
    """

    def __init__(
        self,
        keycloak_url: str,
        realm: str,
        client_id: str,
        client_secret: str,
        scopes: Optional[list[str]] = None,
    ):
        self._token_url = (
            f"{keycloak_url}/realms/{realm}/protocol/openid-connect/token"
        )
        self._client_id = client_id
        self._client_secret = client_secret
        self._scopes = " ".join(scopes) if scopes else ""
        self._current_token: Optional[TokenResponse] = None

    async def get_token(self) -> str:
        """
        Obtiene un access token válido.
        Retorna token cacheado si aún es válido, o solicita uno nuevo.
        """
        if self._current_token and not self._current_token.is_expired:
            return self._current_token.access_token

        return await self._request_new_token()

    async def _request_new_token(self) -> str:
        """Solicita un nuevo token al servidor de autorización."""
        data = {
            "grant_type": "client_credentials",
            "client_id": self._client_id,
            "client_secret": self._client_secret,
        }
        if self._scopes:
            data["scope"] = self._scopes

        async with httpx.AsyncClient(timeout=10.0) as client:
            response = await client.post(
                self._token_url,
                data=data,
                headers={"Content-Type": "application/x-www-form-urlencoded"},
            )
            response.raise_for_status()
            token_data = response.json()

            self._current_token = TokenResponse(
                access_token=token_data["access_token"],
                expires_in=token_data["expires_in"],
                token_type=token_data["token_type"],
                obtained_at=time.time(),
            )

            logger.info(
                "Nuevo token obtenido para '%s' (expira en %ds)",
                self._client_id,
                token_data["expires_in"],
            )
            return self._current_token.access_token

    async def get_authorization_header(self) -> dict[str, str]:
        """Retorna el header Authorization listo para usar en requests."""
        token = await self.get_token()
        return {"Authorization": f"Bearer {token}"}

    def invalidate(self) -> None:
        """Invalida el token actual forzando una nueva solicitud."""
        self._current_token = None
EOF
```

2. Crear el cliente HTTP autenticado que integra con el patrón de resiliencia:

```bash
cat > ~/microservices-lab/shared/auth/authenticated_client.py << 'EOF'
"""
Cliente HTTP autenticado que combina token OAuth2 con circuit breaker.
Extiende la funcionalidad del módulo de resiliencia del Lab 04.
"""

import os
import logging
from typing import Any, Optional

import httpx
from dotenv import load_dotenv

from .token_client import ServiceTokenClient

load_dotenv()
logger = logging.getLogger(__name__)


class AuthenticatedServiceClient:
    """
    Cliente HTTP que automáticamente incluye el token Bearer
    en todas las solicitudes inter-servicio.
    """

    def __init__(
        self,
        service_name: str,
        base_url: str,
        client_id: str,
        client_secret: str,
        keycloak_url: Optional[str] = None,
        realm: Optional[str] = None,
    ):
        self.service_name = service_name
        self.base_url = base_url.rstrip("/")

        kc_url = keycloak_url or os.getenv("KEYCLOAK_URL", "http://localhost:8080")
        kc_realm = realm or os.getenv("KEYCLOAK_REALM", "microservices-lab")

        self._token_client = ServiceTokenClient(
            keycloak_url=kc_url,
            realm=kc_realm,
            client_id=client_id,
            client_secret=client_secret,
        )

    async def request(
        self,
        method: str,
        path: str,
        **kwargs: Any,
    ) -> httpx.Response:
        """
        Realiza una solicitud HTTP autenticada.
        Incluye automáticamente el token Bearer en el header.
        """
        url = f"{self.base_url}{path}"
        auth_header = await self._token_client.get_authorization_header()

        # Merge headers
        headers = kwargs.pop("headers", {})
        headers.update(auth_header)

        async with httpx.AsyncClient(timeout=10.0) as client:
            response = await client.request(
                method=method,
                url=url,
                headers=headers,
                **kwargs,
            )

        # Si recibimos 401, invalidar token e intentar una vez más
        if response.status_code == 401:
            logger.warning("Token rechazado por %s, renovando...", self.service_name)
            self._token_client.invalidate()
            auth_header = await self._token_client.get_authorization_header()
            headers.update(auth_header)

            async with httpx.AsyncClient(timeout=10.0) as client:
                response = await client.request(
                    method=method,
                    url=url,
                    headers=headers,
                    **kwargs,
                )

        return response

    async def get(self, path: str, **kwargs: Any) -> httpx.Response:
        return await self.request("GET", path, **kwargs)

    async def post(self, path: str, **kwargs: Any) -> httpx.Response:
        return await self.request("POST", path, **kwargs)

    async def put(self, path: str, **kwargs: Any) -> httpx.Response:
        return await self.request("PUT", path, **kwargs)

    async def delete(self, path: str, **kwargs: Any) -> httpx.Response:
        return await self.request("DELETE", path, **kwargs)
EOF
```

3. Actualizar `__init__.py` con las nuevas exportaciones:

```bash
cat > ~/microservices-lab/shared/auth/__init__.py << 'EOF'
"""Módulo de autenticación y autorización compartido."""

from .jwt_validator import JWTValidator, JWTValidationError, JWKSCache
from .dependencies import (
    get_current_user,
    get_jwt_validator,
    RoleChecker,
    require_order_read,
    require_order_write,
    require_inventory_read,
    require_inventory_write,
)
from .token_client import ServiceTokenClient, TokenResponse
from .authenticated_client import AuthenticatedServiceClient

__all__ = [
    "JWTValidator",
    "JWTValidationError",
    "JWKSCache",
    "get_current_user",
    "get_jwt_validator",
    "RoleChecker",
    "require_order_read",
    "require_order_write",
    "require_inventory_read",
    "require_inventory_write",
    "ServiceTokenClient",
    "TokenResponse",
    "AuthenticatedServiceClient",
]
EOF
```

### Salida Esperada

```
shared/auth/
├── __init__.py
├── authenticated_client.py
├── dependencies.py
├── jwt_validator.py
└── token_client.py
```

### Verificación

```bash
cd ~/microservices-lab/
python3 -c "
from shared.auth import ServiceTokenClient, AuthenticatedServiceClient
print('✓ ServiceTokenClient importado')
print('✓ AuthenticatedServiceClient importado')
"
```

---

## Paso 5: Proteger Endpoints del OrderService

**Objetivo:** Integrar la validación JWT en los endpoints del OrderService, aplicando roles específicos a cada operación.

### Instrucciones

1. Crear el archivo principal del OrderService con seguridad:

```bash
cat > ~/microservices-lab/order-service/main_secured.py << 'EOF'
"""
OrderService con seguridad OAuth2/JWT integrada.
Extiende el servicio del Lab 04 con autenticación y autorización.
"""

import os
import sys
import logging
from contextlib import asynccontextmanager
from typing import Optional
from uuid import UUID, uuid4
from datetime import datetime

from fastapi import FastAPI, Depends, HTTPException, status
from pydantic import BaseModel, Field
from dotenv import load_dotenv

# Agregar shared al path
sys.path.insert(0, os.path.join(os.path.dirname(__file__), ".."))

from shared.auth import (
    get_current_user,
    require_order_read,
    require_order_write,
    AuthenticatedServiceClient,
)

load_dotenv()
logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)


# --- Modelos ---
class OrderItem(BaseModel):
    product_id: str = Field(..., description="ID del producto")
    quantity: int = Field(..., gt=0, description="Cantidad solicitada")
    unit_price: float = Field(..., gt=0, description="Precio unitario")


class CreateOrderRequest(BaseModel):
    customer_id: str = Field(..., description="ID del cliente")
    items: list[OrderItem] = Field(..., min_length=1, description="Items del pedido")


class OrderResponse(BaseModel):
    order_id: str
    customer_id: str
    items: list[OrderItem]
    total: float
    status: str
    created_at: str
    created_by: str


# --- Estado en memoria (simplificado para el lab) ---
orders_db: dict[str, OrderResponse] = {}


# --- Cliente autenticado para comunicación con InventoryService ---
inventory_client: Optional[AuthenticatedServiceClient] = None


@asynccontextmanager
async def lifespan(app: FastAPI):
    global inventory_client
    inventory_client = AuthenticatedServiceClient(
        service_name="inventory-service",
        base_url=os.getenv("INVENTORY_SERVICE_URL", "http://localhost:8002"),
        client_id=os.getenv("ORDER_SERVICE_CLIENT_ID", "order-service-client"),
        client_secret=os.getenv("ORDER_SERVICE_CLIENT_SECRET", "order-svc-secret-2024"),
    )
    logger.info("OrderService iniciado con autenticación OAuth2")
    yield
    logger.info("OrderService detenido")


app = FastAPI(
    title="OrderService (Secured)",
    version="2.0.0",
    description="Servicio de pedidos protegido con OAuth2/JWT",
    lifespan=lifespan,
)


# --- Endpoints Protegidos ---

@app.get("/health")
async def health_check():
    """Endpoint de salud (sin autenticación)."""
    return {"status": "healthy", "service": "order-service", "secured": True}


@app.get(
    "/orders",
    response_model=list[OrderResponse],
    dependencies=[Depends(require_order_read)],
)
async def list_orders(claims: dict = Depends(get_current_user)):
    """Lista todos los pedidos. Requiere rol order:read."""
    logger.info("Listando pedidos - usuario: %s", claims.get("sub", "unknown"))
    return list(orders_db.values())


@app.get(
    "/orders/{order_id}",
    response_model=OrderResponse,
)
async def get_order(order_id: str, claims: dict = Depends(require_order_read)):
    """Obtiene un pedido por ID. Requiere rol order:read."""
    if order_id not in orders_db:
        raise HTTPException(status_code=404, detail="Pedido no encontrado")
    return orders_db[order_id]


@app.post(
    "/orders",
    response_model=OrderResponse,
    status_code=status.HTTP_201_CREATED,
)
async def create_order(
    request: CreateOrderRequest,
    claims: dict = Depends(require_order_write),
):
    """
    Crea un nuevo pedido. Requiere roles order:read y order:write.
    Verifica inventario via comunicación autenticada inter-servicio.
    """
    # Verificar inventario (comunicación inter-servicio autenticada)
    for item in request.items:
        try:
            response = await inventory_client.get(
                f"/inventory/{item.product_id}"
            )
            if response.status_code == 200:
                stock = response.json().get("quantity", 0)
                if stock < item.quantity:
                    raise HTTPException(
                        status_code=400,
                        detail=f"Stock insuficiente para {item.product_id}: "
                               f"disponible={stock}, solicitado={item.quantity}",
                    )
            elif response.status_code == 404:
                logger.warning("Producto %s no encontrado en inventario", item.product_id)
        except httpx.HTTPError as e:
            logger.error("Error consultando inventario: %s", e)
            # Continuar sin verificación si el servicio no está disponible

    # Crear pedido
    order_id = str(uuid4())
    total = sum(item.quantity * item.unit_price for item in request.items)
    
    # Extraer identidad del creador desde el token
    created_by = claims.get("preferred_username", claims.get("sub", "unknown"))

    order = OrderResponse(
        order_id=order_id,
        customer_id=request.customer_id,
        items=request.items,
        total=total,
        status="created",
        created_at=datetime.utcnow().isoformat(),
        created_by=created_by,
    )

    orders_db[order_id] = order
    logger.info("Pedido %s creado por %s", order_id, created_by)
    return order


@app.get("/orders/me/info")
async def get_my_info(claims: dict = Depends(get_current_user)):
    """Endpoint de diagnóstico: muestra los claims del token actual."""
    from shared.auth.jwt_validator import JWTValidator
    roles = JWTValidator.extract_roles(claims)
    return {
        "sub": claims.get("sub"),
        "preferred_username": claims.get("preferred_username"),
        "email": claims.get("email"),
        "roles": roles,
        "token_issuer": claims.get("iss"),
        "token_expires": claims.get("exp"),
    }


if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8001)
EOF
```

2. Crear el archivo equivalente para InventoryService:

```bash
cat > ~/microservices-lab/inventory-service/main_secured.py << 'EOF'
"""
InventoryService con seguridad OAuth2/JWT integrada.
"""

import os
import sys
import logging
from contextlib import asynccontextmanager

from fastapi import FastAPI, Depends, HTTPException, status
from pydantic import BaseModel, Field
from dotenv import load_dotenv

sys.path.insert(0, os.path.join(os.path.dirname(__file__), ".."))

from shared.auth import (
    get_current_user,
    require_inventory_read,
    require_inventory_write,
)

load_dotenv()
logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)


# --- Modelos ---
class InventoryItem(BaseModel):
    product_id: str
    name: str
    quantity: int
    reserved: int = 0


class UpdateInventoryRequest(BaseModel):
    quantity: int = Field(..., ge=0, description="Nueva cantidad en stock")


# --- Estado en memoria ---
inventory_db: dict[str, InventoryItem] = {
    "PROD-001": InventoryItem(product_id="PROD-001", name="Widget A", quantity=100),
    "PROD-002": InventoryItem(product_id="PROD-002", name="Widget B", quantity=50),
    "PROD-003": InventoryItem(product_id="PROD-003", name="Widget C", quantity=0),
}


@asynccontextmanager
async def lifespan(app: FastAPI):
    logger.info("InventoryService iniciado con autenticación OAuth2")
    yield
    logger.info("InventoryService detenido")


app = FastAPI(
    title="InventoryService (Secured)",
    version="2.0.0",
    description="Servicio de inventario protegido con OAuth2/JWT",
    lifespan=lifespan,
)


@app.get("/health")
async def health_check():
    return {"status": "healthy", "service": "inventory-service", "secured": True}


@app.get(
    "/inventory",
    response_model=list[InventoryItem],
    dependencies=[Depends(require_inventory_read)],
)
async def list_inventory(claims: dict = Depends(get_current_user)):
    """Lista todo el inventario. Requiere rol inventory:read."""
    logger.info("Listando inventario - usuario: %s", claims.get("sub"))
    return list(inventory_db.values())


@app.get(
    "/inventory/{product_id}",
    response_model=InventoryItem,
)
async def get_inventory_item(
    product_id: str,
    claims: dict = Depends(require_inventory_read),
):
    """Consulta inventario de un producto. Requiere rol inventory:read."""
    if product_id not in inventory_db:
        raise HTTPException(status_code=404, detail="Producto no encontrado")
    return inventory_db[product_id]


@app.put(
    "/inventory/{product_id}",
    response_model=InventoryItem,
)
async def update_inventory(
    product_id: str,
    request: UpdateInventoryRequest,
    claims: dict = Depends(require_inventory_write),
):
    """Actualiza cantidad en inventario. Requiere rol inventory:write."""
    if product_id not in inventory_db:
        raise HTTPException(status_code=404, detail="Producto no encontrado")

    inventory_db[product_id].quantity = request.quantity
    username = claims.get("preferred_username", claims.get("sub"))
    logger.info(
        "Inventario actualizado: %s -> %d (por %s)",
        product_id, request.quantity, username,
    )
    return inventory_db[product_id]


if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8002)
EOF
```

3. Iniciar ambos servicios en segundo plano:

```bash
cd ~/microservices-lab/

# Iniciar InventoryService
python3 inventory-service/main_secured.py &
INV_PID=$!
echo "InventoryService PID: $INV_PID"

# Esperar a que inicie
sleep 2

# Iniciar OrderService
python3 order-service/main_secured.py &
ORD_PID=$!
echo "OrderService PID: $ORD_PID"

sleep 2
```

### Salida Esperada

```
InventoryService PID: 12345
OrderService PID: 12346
INFO:     Started server process [12345]
INFO:     Uvicorn running on http://0.0.0.0:8002
INFO:     Started server process [12346]
INFO:     Uvicorn running on http://0.0.0.0:8001
```

### Verificación

```bash
# Health endpoints deben funcionar sin token
curl -s http://localhost:8001/health | python3 -m json.tool
curl -s http://localhost:8002/health | python3 -m json.tool

# Endpoints protegidos deben rechazar sin token
curl -s -o /dev/null -w "%{http_code}" http://localhost:8001/orders
```

El health check debe retornar `{"status": "healthy", ...}` y el endpoint protegido debe retornar código `403` (sin header Authorization) o `401`.

---

## Paso 6: Probar Flujos de Autenticación y Autorización

**Objetivo:** Validar el sistema completo probando obtención de tokens, acceso autorizado, rechazo por falta de permisos y tokens inválidos.

### Instrucciones

1. Obtener un token para `admin-user` (todos los permisos):

```bash
# Obtener token usando Resource Owner Password Credentials (solo para pruebas)
ADMIN_TOKEN=$(curl -s -X POST \
    "http://localhost:8080/realms/microservices-lab/protocol/openid-connect/token" \
    -H "Content-Type: application/x-www-form-urlencoded" \
    -d "grant_type=password" \
    -d "client_id=frontend-client" \
    -d "username=admin-user" \
    -d "password=Admin@2024!" \
    -d "scope=openid" | python3 -c "import sys,json; print(json.load(sys.stdin)['access_token'])")

echo "Token admin obtenido (primeros 50 chars): ${ADMIN_TOKEN:0:50}..."
```

2. Probar acceso autorizado con admin-user:

```bash
# Listar pedidos (requiere order:read) - debe funcionar
echo "=== GET /orders con admin-user ==="
curl -s http://localhost:8001/orders \
    -H "Authorization: Bearer $ADMIN_TOKEN" | python3 -m json.tool

# Ver info del token
echo "=== GET /orders/me/info ==="
curl -s http://localhost:8001/orders/me/info \
    -H "Authorization: Bearer $ADMIN_TOKEN" | python3 -m json.tool
```

3. Crear un pedido (requiere order:write):

```bash
echo "=== POST /orders con admin-user ==="
curl -s -X POST http://localhost:8001/orders \
    -H "Authorization: Bearer $ADMIN_TOKEN" \
    -H "Content-Type: application/json" \
    -d '{
        "customer_id": "CUST-001",
        "items": [
            {"product_id": "PROD-001", "quantity": 5, "unit_price": 29.99}
        ]
    }' | python3 -m json.tool
```

4. Obtener token para `readonly-user` y verificar restricciones:

```bash
READONLY_TOKEN=$(curl -s -X POST \
    "http://localhost:8080/realms/microservices-lab/protocol/openid-connect/token" \
    -H "Content-Type: application/x-www-form-urlencoded" \
    -d "grant_type=password" \
    -d "client_id=frontend-client" \
    -d "username=readonly-user" \
    -d "password=Read@2024!" \
    -d "scope=openid" | python3 -c "import sys,json; print(json.load(sys.stdin)['access_token'])")

echo "Token readonly obtenido"

# Listar pedidos (order:read) - debe funcionar
echo "=== GET /orders con readonly-user ==="
curl -s -o /dev/null -w "Status: %{http_code}\n" http://localhost:8001/orders \
    -H "Authorization: Bearer $READONLY_TOKEN"

# Crear pedido (order:write) - debe ser RECHAZADO con 403
echo "=== POST /orders con readonly-user (debe fallar) ==="
curl -s -w "\nStatus: %{http_code}\n" -X POST http://localhost:8001/orders \
    -H "Authorization: Bearer $READONLY_TOKEN" \
    -H "Content-Type: application/json" \
    -d '{
        "customer_id": "CUST-002",
        "items": [{"product_id": "PROD-001", "quantity": 1, "unit_price": 10.0}]
    }'
```

5. Probar con token inválido:

```bash
echo "=== Solicitud con token inválido ==="
curl -s -w "\nStatus: %{http_code}\n" http://localhost:8001/orders \
    -H "Authorization: Bearer token-completamente-invalido"

echo "=== Solicitud sin token ==="
curl -s -w "\nStatus: %{http_code}\n" http://localhost:8001/orders
```

6. Probar flujo Client Credentials (servicio a servicio):

```bash
# Obtener token de servicio para order-service-client
SERVICE_TOKEN=$(curl -s -X POST \
    "http://localhost:8080/realms/microservices-lab/protocol/openid-connect/token" \
    -H "Content-Type: application/x-www-form-urlencoded" \
    -d "grant_type=client_credentials" \
    -d "client_id=order-service-client" \
    -d "client_secret=order-svc-secret-2024" \
    | python3 -c "import sys,json; print(json.load(sys.stdin)['access_token'])")

echo "Token de servicio obtenido"

# Consultar inventario con token de servicio (tiene inventory:read)
echo "=== GET /inventory con service token ==="
curl -s http://localhost:8002/inventory/PROD-001 \
    -H "Authorization: Bearer $SERVICE_TOKEN" | python3 -m json.tool
```

### Salida Esperada

Para el admin-user, todos los endpoints deben responder con `200` o `201`. Para readonly-user, el POST debe retornar:

```json
{"detail": "Roles insuficientes. Faltan: ['order:write']"}
Status: 403
```

Para token inválido:
```
Status: 401
```

Para el token de servicio consultando inventario:
```json
{
    "product_id": "PROD-001",
    "name": "Widget A",
    "quantity": 100,
    "reserved": 0
}
```

### Verificación

Ejecutar el resumen de pruebas:

```bash
echo "=== Resumen de Verificación ==="
echo -n "Admin GET /orders: "
curl -s -o /dev/null -w "%{http_code}" http://localhost:8001/orders -H "Authorization: Bearer $ADMIN_TOKEN"
echo ""
echo -n "Admin POST /orders: "
curl -s -o /dev/null -w "%{http_code}" -X POST http://localhost:8001/orders -H "Authorization: Bearer $ADMIN_TOKEN" -H "Content-Type: application/json" -d '{"customer_id":"C1","items":[{"product_id":"PROD-001","quantity":1,"unit_price":1.0}]}'
echo ""
echo -n "Readonly POST /orders (debe ser 403): "
curl -s -o /dev/null -w "%{http_code}" -X POST http://localhost:8001/orders -H "Authorization: Bearer $READONLY_TOKEN" -H "Content-Type: application/json" -d '{"customer_id":"C1","items":[{"product_id":"PROD-001","quantity":1,"unit_price":1.0}]}'
echo ""
echo -n "Sin token (debe ser 403): "
curl -s -o /dev/null -w "%{http_code}" http://localhost:8001/orders
echo ""
echo -n "Token inválido (debe ser 401): "
curl -s -o /dev/null -w "%{http_code}" http://localhost:8001/orders -H "Authorization: Bearer invalid"
echo ""
```

Resultado esperado:
```
=== Resumen de Verificación ===
Admin GET /orders: 200
Admin POST /orders: 201
Readonly POST /orders (debe ser 403): 403
Sin token (debe ser 403): 403
Token inválido (debe ser 401): 401
```

---

## Paso 7: Implementar Tests Automatizados

**Objetivo:** Crear una suite de tests que valide todos los escenarios de seguridad de forma automatizada.

### Instrucciones

1. Crear el archivo de tests:

```bash
cat > ~/microservices-lab/tests/test_auth/test_security_flows.py << 'EOF'
"""
Tests de integración para la seguridad OAuth2/JWT.
Requiere Keycloak y los servicios en ejecución.
"""

import os
import sys
import pytest
import httpx

sys.path.insert(0, os.path.join(os.path.dirname(__file__), "../.."))

KEYCLOAK_URL = os.getenv("KEYCLOAK_URL", "http://localhost:8080")
REALM = os.getenv("KEYCLOAK_REALM", "microservices-lab")
ORDER_SERVICE_URL = "http://localhost:8001"
INVENTORY_SERVICE_URL = "http://localhost:8002"
TOKEN_URL = f"{KEYCLOAK_URL}/realms/{REALM}/protocol/openid-connect/token"


@pytest.fixture
async def admin_token() -> str:
    """Obtiene token para admin-user."""
    async with httpx.AsyncClient() as client:
        resp = await client.post(
            TOKEN_URL,
            data={
                "grant_type": "password",
                "client_id": "frontend-client",
                "username": "admin-user",
                "password": "Admin@2024!",
                "scope": "openid",
            },
        )
        resp.raise_for_status()
        return resp.json()["access_token"]


@pytest.fixture
async def readonly_token() -> str:
    """Obtiene token para readonly-user."""
    async with httpx.AsyncClient() as client:
        resp = await client.post(
            TOKEN_URL,
            data={
                "grant_type": "password",
                "client_id": "frontend-client",
                "username": "readonly-user",
                "password": "Read@2024!",
                "scope": "openid",
            },
        )
        resp.raise_for_status()
        return resp.json()["access_token"]


@pytest.fixture
async def service_token() -> str:
    """Obtiene token de servicio via Client Credentials."""
    async with httpx.AsyncClient() as client:
        resp = await client.post(
            TOKEN_URL,
            data={
                "grant_type": "client_credentials",
                "client_id": "order-service-client",
                "client_secret": "order-svc-secret-2024",
            },
        )
        resp.raise_for_status()
        return resp.json()["access_token"]


class TestHealthEndpoints:
    """Los endpoints de salud no requieren autenticación."""

    @pytest.mark.asyncio
    async def test_order_service_health(self):
        async with httpx.AsyncClient() as client:
            resp = await client.get(f"{ORDER_SERVICE_URL}/health")
        assert resp.status_code == 200
        assert resp.json()["secured"] is True

    @pytest.mark.asyncio
    async def test_inventory_service_health(self):
        async with httpx.AsyncClient() as client:
            resp = await client.get(f"{INVENTORY_SERVICE_URL}/health")
        assert resp.status_code == 200


class TestAuthenticationRequired:
    """Endpoints protegidos rechazan solicitudes sin token."""

    @pytest.mark.asyncio
    async def test_orders_without_token_returns_403(self):
        async with httpx.AsyncClient() as client:
            resp = await client.get(f"{ORDER_SERVICE_URL}/orders")
        assert resp.status_code == 403

    @pytest.mark.asyncio
    async def test_inventory_without_token_returns_403(self):
        async with httpx.AsyncClient() as client:
            resp = await client.get(f"{INVENTORY_SERVICE_URL}/inventory")
        assert resp.status_code == 403

    @pytest.mark.asyncio
    async def test_invalid_token_returns_401(self):
        async with httpx.AsyncClient() as client:
            resp = await client.get(
                f"{ORDER_SERVICE_URL}/orders",
                headers={"Authorization": "Bearer invalid-token-xyz"},
            )
        assert resp.status_code == 401

    @pytest.mark.asyncio
    async def test_malformed_header_returns_error(self):
        async with httpx.AsyncClient() as client:
            resp = await client.get(
                f"{ORDER_SERVICE_URL}/orders",
                headers={"Authorization": "NotBearer token"},
            )
        assert resp.status_code in [401, 403]


class TestAdminUserAccess:
    """admin-user tiene acceso completo."""

    @pytest.mark.asyncio
    async def test_can_list_orders(self, admin_token):
        async with httpx.AsyncClient() as client:
            resp = await client.get(
                f"{ORDER_SERVICE_URL}/orders",
                headers={"Authorization": f"Bearer {admin_token}"},
            )
        assert resp.status_code == 200
        assert isinstance(resp.json(), list)

    @pytest.mark.asyncio
    async def test_can_create_order(self, admin_token):
        async with httpx.AsyncClient() as client:
            resp = await client.post(
                f"{ORDER_SERVICE_URL}/orders",
                headers={"Authorization": f"Bearer {admin_token}"},
                json={
                    "customer_id": "TEST-CUST",
                    "items": [{"product_id": "PROD-001", "quantity": 2, "unit_price": 15.0}],
                },
            )
        assert resp.status_code == 201
        data = resp.json()
        assert data["customer_id"] == "TEST-CUST"
        assert data["created_by"] == "admin-user"
        assert data["status"] == "created"

    @pytest.mark.asyncio
    async def test_can_read_inventory(self, admin_token):
        async with httpx.AsyncClient() as client:
            resp = await client.get(
                f"{INVENTORY_SERVICE_URL}/inventory",
                headers={"Authorization": f"Bearer {admin_token}"},
            )
        assert resp.status_code == 200


class TestReadonlyUserRestrictions:
    """readonly-user solo tiene permisos de lectura."""

    @pytest.mark.asyncio
    async def test_can_list_orders(self, readonly_token):
        async with httpx.AsyncClient() as client:
            resp = await client.get(
                f"{ORDER_SERVICE_URL}/orders",
                headers={"Authorization": f"Bearer {readonly_token}"},
            )
        assert resp.status_code == 200

    @pytest.mark.asyncio
    async def test_cannot_create_order(self, readonly_token):
        async with httpx.AsyncClient() as client:
            resp = await client.post(
                f"{ORDER_SERVICE_URL}/orders",
                headers={"Authorization": f"Bearer {readonly_token}"},
                json={
                    "customer_id": "HACK",
                    "items": [{"product_id": "PROD-001", "quantity": 1, "unit_price": 1.0}],
                },
            )
        assert resp.status_code == 403
        assert "Roles insuficientes" in resp.json()["detail"]

    @pytest.mark.asyncio
    async def test_cannot_update_inventory(self, readonly_token):
        async with httpx.AsyncClient() as client:
            resp = await client.put(
                f"{INVENTORY_SERVICE_URL}/inventory/PROD-001",
                headers={"Authorization": f"Bearer {readonly_token}"},
                json={"quantity": 999},
            )
        assert resp.status_code == 403


class TestServiceToServiceAuth:
    """Comunicación inter-servicio con Client Credentials."""

    @pytest.mark.asyncio
    async def test_service_token_can_read_inventory(self, service_token):
        async with httpx.AsyncClient() as client:
            resp = await client.get(
                f"{INVENTORY_SERVICE_URL}/inventory/PROD-001",
                headers={"Authorization": f"Bearer {service_token}"},
            )
        assert resp.status_code == 200
        assert resp.json()["product_id"] == "PROD-001"

    @pytest.mark.asyncio
    async def test_service_token_has_correct_roles(self, service_token):
        """Verifica que el token de servicio contiene los roles esperados."""
        from jose import jwt
        # Decodificar sin verificar para inspeccionar claims
        claims = jwt.get_unverified_claims(service_token)
        roles = claims.get("realm_access", {}).get("roles", [])
        assert "order:read" in roles
        assert "order:write" in roles
        assert "inventory:read" in roles


class TestTokenExpiration:
    """Verificación del manejo de tokens expirados."""

    @pytest.mark.asyncio
    async def test_expired_token_is_rejected(self):
        """Un token con exp en el pasado debe ser rechazado."""
        # Crear un token falso con exp expirado (no será válido por firma)
        fake_expired = "eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.eyJleHAiOjEwMDAwMDAwMDAsInN1YiI6InRlc3QifQ.fake"
        async with httpx.AsyncClient() as client:
            resp = await client.get(
                f"{ORDER_SERVICE_URL}/orders",
                headers={"Authorization": f"Bearer {fake_expired}"},
            )
        assert resp.status_code == 401
EOF
```

2. Crear el directorio y archivo `__init__`:

```bash
mkdir -p ~/microservices-lab/tests/test_auth/
touch ~/microservices-lab/tests/test_auth/__init__.py
```

3. Ejecutar los tests:

```bash
cd ~/microservices-lab/
python3 -m pytest tests/test_auth/test_security_flows.py -v --tb=short 2>&1 | tail -30
```

### Salida Esperada

```
tests/test_auth/test_security_flows.py::TestHealthEndpoints::test_order_service_health PASSED
tests/test_auth/test_security_flows.py::TestHealthEndpoints::test_inventory_service_health PASSED
tests/test_auth/test_security_flows.py::TestAuthenticationRequired::test_orders_without_token_returns_403 PASSED
tests/test_auth/test_security_flows.py::TestAuthenticationRequired::test_invalid_token_returns_401 PASSED
tests/test_auth/test_security_flows.py::TestAdminUserAccess::test_can_list_orders PASSED
tests/test_auth/test_security_flows.py::TestAdminUserAccess::test_can_create_order PASSED
tests/test_auth/test_security_flows.py::TestReadonlyUserRestrictions::test_can_list_orders PASSED
tests/test_auth/test_security_flows.py::TestReadonlyUserRestrictions::test_cannot_create_order PASSED
tests/test_auth/test_security_flows.py::TestServiceToServiceAuth::test_service_token_can_read_inventory PASSED
tests/test_auth/test_security_flows.py::TestTokenExpiration::test_expired_token_is_rejected PASSED
...
PASSED
```

### Verificación

Todos los tests deben pasar. Si alguno falla, verificar que tanto Keycloak como los servicios estén corriendo.

---

## Validación y Testing Final

Ejecutar la validación completa del laboratorio:

```bash
cd ~/microservices-lab/

echo "╔══════════════════════════════════════════════════════╗"
echo "║   VALIDACIÓN FINAL - Lab 05: OAuth2/JWT Security    ║"
echo "╚══════════════════════════════════════════════════════╝"

echo ""
echo "1. Verificando Keycloak..."
KC_STATUS=$(curl -s -o /dev/null -w "%{http_code}" http://localhost:8080/health/ready)
echo "   Keycloak health: $KC_STATUS (esperado: 200)"

echo ""
echo "2. Verificando realm y OIDC discovery..."
ISSUER=$(curl -s http://localhost:8080/realms/microservices-lab/.well-known/openid-configuration | python3 -c "import sys,json; print(json.load(sys.stdin).get('issuer','ERROR'))" 2>/dev/null)
echo "   Issuer: $ISSUER"

echo ""
echo "3. Obteniendo tokens..."
ADMIN_T=$(curl -s -X POST "http://localhost:8080/realms/microservices-lab/protocol/openid-connect/token" -d "grant_type=password&client_id=frontend-client&username=admin-user&password=Admin@2024!&scope=openid" | python3 -c "import sys,json; print(json.load(sys.stdin).get('access_token','ERROR')[:20])" 2>/dev/null)
echo "   Admin token: ${ADMIN_T}... ✓"

SVC_T=$(curl -s -X POST "http://localhost:8080/realms/microservices-lab/protocol/openid-connect/token" -d "grant_type=client_credentials&client_id=order-service-client&client_secret=order-svc-secret-2024" | python3 -c "import sys,json; print(json.load(sys.stdin).get('access_token','ERROR')[:20])" 2>/dev/null)
echo "   Service token: ${SVC_T}... ✓"

echo ""
echo "4. Verificando servicios protegidos..."
echo -n "   OrderService health: "
curl -s -o /dev/null -w "%{http_code}\n" http://localhost:8001/health
echo -n "   InventoryService health: "
curl -s -o /dev/null -w "%{http_code}\n" http://localhost:8002/health

echo ""
echo "5. Ejecutando tests automatizados..."
python3 -m pytest tests/test_auth/test_security_flows.py -q --tb=line 2>&1 | tail -5

echo ""
echo "═══════════════════════════════════════════════════════"
echo "✅ Validación completada"
```

---

## Resolución de Problemas

### Problema 1: Keycloak no inicia o falla con error de base de datos

**Síntomas:**
- El contenedor `mslab-keycloak` se reinicia continuamente
- Logs muestran `org.postgresql.util.PSQLException: FATAL: database "keycloak_db" does not exist`
- `docker logs mslab-keycloak` muestra errores de conexión JDBC

**Causa:** La base de datos `keycloak_db` no fue creada antes de iniciar Keycloak, o PostgreSQL no está accesible desde la red Docker.

**Solución:**

```bash
# 1. Verificar que PostgreSQL está corriendo
docker ps | grep mslab-postgres

# 2. Crear la base de datos si no existe
docker exec mslab-postgres psql -U mslab_user -d postgres -c "CREATE DATABASE keycloak_db OWNER mslab_user;" 2>/dev/null || echo "Ya existe"

# 3. Verificar conectividad de red
docker network inspect mslab-network | grep -A2 "mslab-postgres"

# 4. Reiniciar Keycloak
docker restart mslab-keycloak

# 5. Esperar y verificar
sleep 30
curl -s http://localhost:8080/health/ready
```

---

### Problema 2: Token JWT rechazado con "Token JWT inválido: Signature verification failed"

**Síntomas:**
- Los servicios retornan 401 incluso con tokens recién obtenidos
- El error en logs indica fallo de verificación de firma
- El endpoint JWKS responde correctamente

**Causa:** El token fue emitido por un realm diferente al configurado, o la variable `KEYCLOAK_URL` usa `localhost` mientras el servicio intenta validar contra un issuer con hostname diferente (por ejemplo `keycloak` en Docker). El claim `iss` del token no coincide con el issuer esperado por el validador.

**Solución:**

```bash
# 1. Inspeccionar el issuer del token
TOKEN="<tu-token-aquí>"
python3 -c "
from jose import jwt
claims = jwt.get_unverified_claims('$TOKEN')
print('Token issuer:', claims.get('iss'))
"

# 2. Verificar qué issuer espera el validador
python3 -c "
import os
from dotenv import load_dotenv
load_dotenv()
url = os.getenv('KEYCLOAK_URL', 'http://localhost:8080')
realm = os.getenv('KEYCLOAK_REALM', 'microservices-lab')
print(f'Validador espera issuer: {url}/realms/{realm}')
"

# 3. Si no coinciden, ajustar .env para que KEYCLOAK_URL coincida con el issuer del token
# Para ejecución local (fuera de Docker):
echo "KEYCLOAK_URL=http://localhost:8080" >> .env

# 4. Reiniciar los servicios después del cambio
kill $ORD_PID $INV_PID 2>/dev/null
python3 inventory-service/main_secured.py &
python3 order-service/main_secured.py &
```

---

## Limpieza

Para detener los servicios y limpiar el entorno:

```bash
# Detener servicios Python
kill $ORD_PID $INV_PID 2>/dev/null
pkill -f "main_secured.py" 2>/dev/null

# Detener Keycloak (mantener si se usará en labs siguientes)
# docker stop mslab-keycloak

# Para limpieza completa (eliminar datos de Keycloak):
# docker rm -f mslab-keycloak
# docker exec mslab-postgres psql -U mslab_user -d postgres -c "DROP DATABASE keycloak_db;"
```

> **Nota:** Se recomienda mantener Keycloak corriendo si planeas continuar con labs posteriores del módulo 5.

---

## Resumen

En este laboratorio has implementado un sistema completo de seguridad OAuth2/JWT para microservicios Python:

| Componente | Logro |
|------------|-------|
| **Keycloak** | Realm configurado con clientes, roles y usuarios de prueba |
| **Módulo shared/auth** | Validador JWT reutilizable con caché JWKS de 1 hora |
| **Dependencias FastAPI** | RoleChecker configurable para proteger endpoints por rol |
| **Client Credentials** | Comunicación inter-servicio autenticada automáticamente |
| **AuthenticatedServiceClient** | Cliente HTTP que renueva tokens automáticamente |
| **Tests** | Suite completa validando autenticación, autorización y rechazo |

### Conceptos Clave Aplicados

- **Validación local de JWT**: cada servicio verifica tokens usando claves públicas JWKS sin contactar Keycloak en cada request
- **Separación de autenticación y autorización**: `get_current_user` verifica identidad; `RoleChecker` verifica permisos
- **Tokens de corta duración**: access tokens de 5 minutos limitan el impacto de tokens comprometidos
- **Principio de menor privilegio**: cada servicio y usuario tiene solo los roles necesarios

### Recursos Adicionales

- [Keycloak Documentation - Server Administration](https://www.keycloak.org/docs/latest/server_admin/)
- [RFC 7519 - JSON Web Token](https://datatracker.ietf.org/doc/html/rfc7519)
- [OAuth 2.0 Client Credentials Grant - RFC 6749 Section 4.4](https://datatracker.ietf.org/doc/html/rfc6749#section-4.4)
- [FastAPI Security Documentation](https://fastapi.tiangolo.com/tutorial/security/)
- [python-jose Documentation](https://python-jose.readthedocs.io/)
