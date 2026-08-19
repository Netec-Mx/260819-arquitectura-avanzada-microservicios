# Arquitectura avanzada de Microservicios

Profundizar en diseño, implementación y operación de microservicios con Python, cubriendo patrones avanzados, resiliencia, seguridad, observabilidad y despliegue en Kubernetes para sistemas críticos cloud-native.

## Estructura

- `CapituloXX/README.md`: guía de laboratorio por capítulo.

## Lista de laboratorios

### Capítulo 1

- [4 Laboratorio: prototipo de servicio con FastAPI y pruebas contractuales](Capitulo01/README.md#4-laboratorio-prototipo-de-servicio-con-fastapi-y-pruebas-contractuales)
  - Descripción: Desarrollar un prototipo de servicio con FastAPI y realizar pruebas contractuales, aplicando los principios de diseño, límites de contexto y modularidad abordados en el capítulo.
  - Duración estimada: 42 min

### Capítulo 2

- [4 Laboratorio: integrar microservicios Python con Kafka y garantizar idempotencia](Capitulo02/README.md#4-laboratorio-integrar-microservicios-python-con-kafka-y-garantizar-idempotencia)
  - Descripción: Integrar microservicios Python mediante Kafka y aplicar mecanismos de idempotencia para controlar la entrega y el procesamiento de eventos.
  - Duración estimada: 57 min

### Capítulo 3

- [4 Laboratorio: implementar Event Store y proyecciones en Python](Capitulo03/README.md#4-laboratorio-implementar-event-store-y-proyecciones-en-python)
  - Descripción: Implementar un Event Store y proyecciones en Python para aplicar CQRS y Event Sourcing y mantener consistencia eventual.
  - Duración estimada: 66 min

### Capítulo 4

- [4 Laboratorio: aplicar circuit breaker y retries en servicios Python](Capitulo04/README.md#4-laboratorio-aplicar-circuit-breaker-y-retries-en-servicios-python)
  - Descripción: Aplicar circuit breaker y retries en servicios Python para mejorar la resiliencia y mitigar puntos de fallo.
  - Duración estimada: 66 min

### Capítulo 5

- [4 Laboratorio: proteger APIs Python con OAuth2 y JWT usando Keycloak](Capitulo05/README.md#4-laboratorio-proteger-apis-python-con-oauth2-y-jwt-usando-keycloak)
  - Descripción: Proteger APIs Python mediante OAuth2 y JWT usando Keycloak para implementar autenticación, autorización y control de acceso seguro.
  - Duración estimada: 66 min

### Capítulo 6

- [4 Laboratorio: desplegar Gateway y políticas para servicios Python](Capitulo06/README.md#4-laboratorio-desplegar-gateway-y-políticas-para-servicios-python)
  - Descripción: Desplegar un Gateway y configurar políticas para servicios Python, aplicando enrutamiento, filtros y controles en el edge.
  - Duración estimada: 66 min

### Capítulo 7

- [4 Laboratorio: integrar Vault con servicios Python para gestión de secretos](Capitulo07/README.md#4-laboratorio-integrar-vault-con-servicios-python-para-gestión-de-secretos)
  - Descripción: Integrar HashiCorp Vault con servicios Python para centralizar y proteger la gestión de secretos en los despliegues.
  - Duración estimada: 66 min

### Capítulo 8

- [4 Laboratorio: instrumentar servicios Python con OpenTelemetry y visualizar métricas](Capitulo08/README.md#4-laboratorio-instrumentar-servicios-python-con-opentelemetry-y-visualizar-métricas)
  - Descripción: Instrumentar servicios Python con OpenTelemetry y visualizar métricas para observar el comportamiento distribuido y apoyar el diagnóstico de latencia y fallos.
  - Duración estimada: 66 min

### Capítulo 9

- [4 Laboratorio: construir imágenes optimizadas y empaquetar con Helm](Capitulo09/README.md#4-laboratorio-construir-imágenes-optimizadas-y-empaquetar-con-helm)
  - Descripción: Construir imágenes optimizadas para aplicaciones Python y empaquetarlas con Helm, aplicando prácticas de construcción, gestión y despliegue descritas en el capítulo.
  - Duración estimada: 66 min

### Capítulo 10

- [4 Laboratorio: desplegar microservicios Python en Kubernetes con Istio](Capitulo10/README.md#4-laboratorio-desplegar-microservicios-python-en-kubernetes-con-istio)
  - Descripción: Desplegar microservicios Python en Kubernetes con Istio para aplicar orquestación, enrutamiento, seguridad mutua y observabilidad mediante service mesh.
  - Duración estimada: 66 min

### Capítulo 11

- [4 Laboratorio: pipeline completo desde commit hasta despliegue en k8s](Capitulo11/README.md#4-laboratorio-pipeline-completo-desde-commit-hasta-despliegue-en-k8s)
  - Descripción: Construir un pipeline completo desde el commit hasta el despliegue en k8s, integrando pruebas, controles de seguridad y automatización de despliegues reproducibles.
  - Duración estimada: 66 min

### Capítulo 12

- [4 Laboratorio: diseñar y ejecutar pruebas de carga y ataques controlados](Capitulo12/README.md#4-laboratorio-diseñar-y-ejecutar-pruebas-de-carga-y-ataques-controlados)
  - Descripción: Diseñar y ejecutar pruebas de carga y ataques controlados para validar propiedades no funcionales y estrategias de recuperación ante fallos en microservicios.
  - Duración estimada: 66 min

## Flujo de colaboración

- Trabajar en `changes_course`.
- Crear Pull Request hacia `main`.
- Merge por `Squash and merge`.
