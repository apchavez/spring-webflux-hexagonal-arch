# Reactive Customer Service API

[![CI](https://github.com/apchavez/reactive-customer-service/actions/workflows/ci.yml/badge.svg)](https://github.com/apchavez/reactive-customer-service/actions/workflows/ci.yml)

API REST reactiva desarrollada con **Spring Boot WebFlux** para gestionar clientes (CRUD completo) siguiendo **arquitectura hexagonal (Ports & Adapters)**.

## Tecnologías

| Categoría | Tecnología |
|---|---|
| Lenguaje / Runtime | Java 21, Spring Boot 3.5.3 |
| Reactividad | Spring WebFlux (Mono / Flux), Spring Data R2DBC |
| Base de datos | H2 (perfil dev) / PostgreSQL 16 (perfil prod) |
| Seguridad | Spring Security (headers, CORS, rate limiting) |
| Observabilidad | Spring Boot Actuator, SLF4J + Logback, X-Request-Id |
| Documentación API | Springdoc OpenAPI 2 (Swagger UI) |
| Build | Gradle 8, JaCoCo (cobertura ≥ 80 % en dominio y aplicación) |
| Calidad de código | ArchUnit, SonarQube |
| Contenerización | Docker (multistage build) + docker-compose |
| Orquestación | Kubernetes (manifests en `k8s/`) |
| CI/CD | GitHub Actions → ghcr.io |

---

## Arquitectura

Hexagonal (Ports & Adapters) con tres capas bien delimitadas:

```
src/main/java/com/apchavez/customers
├── domain
│   ├── model          Customer (record con invariantes), CustomerState
│   ├── exception      Excepciones de dominio tipadas
│   ├── port           CustomerRepositoryPort (interfaz — el contrato)
│   └── service        CustomerDomainService (lógica de negocio pura)
├── application
│   └── CustomerApplicationService  (orquestación, logging de auditoría, @Transactional)
└── infrastructure
    ├── config         Security, RateLimiting, RequestLogging, OpenApi, Startup
    ├── mapper         CustomerMapper (DTO ↔ Domain ↔ Entity)
    ├── persistence    CustomerEntity, CustomerR2dbcRepository, CustomerPersistenceAdapter
    └── web            CustomerController, DTOs (Request/Update/Response), GlobalExceptionHandler
```

**Regla de dependencias:** `infrastructure` → `application` → `domain`  
El dominio no conoce las capas externas. Verificado automáticamente por `ArchitectureTest` (ArchUnit).

---

## Ejecutar localmente (H2 en memoria)

```bash
./gradlew bootRun
```

- App: `http://localhost:8080`
- Swagger UI: `http://localhost:8080/swagger-ui.html`

## Ejecutar con Docker (PostgreSQL)

```bash
# Opcional: personalizar credenciales
cp .env.example .env
# editar .env con los valores deseados

docker compose up --build
```

> El script `docker/postgres-init.sql` inicializa el schema de PostgreSQL automáticamente en el primer arranque.

---

## Endpoints

Base path: `/api/v1/customers`

| Método | Ruta | Descripción | Respuestas |
|---|---|---|---|
| `POST` | `/` | Crear cliente | `201`, `400`, `422` |
| `GET` | `/active` | Listar clientes activos | `200` |
| `GET` | `/{id}` | Buscar por ID | `200`, `404` |
| `PUT` | `/{id}` | Actualizar cliente completo | `200`, `400`, `404`, `422` |
| `DELETE` | `/{id}` | Eliminar cliente | `204`, `404` |

La documentación interactiva completa (con ejemplos de request/response y todos los códigos de error) está disponible en Swagger UI.

### Ejemplos rápidos

```bash
# Crear cliente
curl -X POST http://localhost:8080/api/v1/customers \
  -H "Content-Type: application/json" \
  -d '{"nombre":"Alex","apellido":"Prieto","estado":"ACTIVE","edad":30}'

# Listar activos
curl http://localhost:8080/api/v1/customers/active

# Buscar por ID
curl http://localhost:8080/api/v1/customers/1

# Actualizar
curl -X PUT http://localhost:8080/api/v1/customers/1 \
  -H "Content-Type: application/json" \
  -d '{"nombre":"Alexander","apellido":"Prieto Chavez","estado":"INACTIVE","edad":31}'

# Eliminar
curl -X DELETE http://localhost:8080/api/v1/customers/1
```

---

## Tests

```bash
./gradlew test          # ejecuta todos los tests + JaCoCo
```

| Tipo | Clase | Descripción |
|---|---|---|
| Modelo de dominio — unit + property-based (jqwik) | `CustomerDomainTest` | Invariantes del record `Customer` |
| Serialización JSON — property-based | `CustomerResponseDTOSerializationTest` | Round-trip sin pérdida de datos |
| Servicio de dominio — unit | `CustomerDomainServiceTest` | Lógica de negocio (create/find/update/delete) |
| Servicio de aplicación — unit | `CustomerApplicationServiceTest` | Orquestación de casos de uso |
| Adaptador de persistencia — `@DataR2dbcTest` | `CustomerPersistenceAdapterTest` | Puerto de persistencia con H2 real |
| Controlador REST — integración completa | `CustomerControllerIntegrationTest` | Todos los endpoints y códigos de respuesta |
| Rate limiter — unit | `RateLimitingFilterTest` | Límite por IP y aislamiento entre IPs |
| Actuator probes | `ActuatorHealthTest` | Liveness/Readiness |
| Arquitectura hexagonal — ArchUnit | `ArchitectureTest` | 4 reglas de dependencia |

---

## CI/CD

El pipeline de GitHub Actions (`.github/workflows/ci.yml`) ejecuta tres jobs en cada push:

| Job | Disparador | Qué hace |
|---|---|---|
| `test` | Todo push / PR | Compila, ejecuta tests, JaCoCo ≥ 80 %, análisis SonarQube (si `SONAR_HOST_URL` está configurado) |
| `k8s-validate` | Todo push / PR | Valida los manifests con **kubeconform** |
| `docker` | Push a `main` (tras `test`) | Build + push a **ghcr.io** con tags `latest` y `sha-XXXXXXX` |

La imagen publicada queda disponible en:

```
ghcr.io/apchavez/reactive-customer-service:latest
ghcr.io/apchavez/reactive-customer-service:sha-abc1234
```

### SonarQube (opcional)

Para activar el análisis de SonarQube en CI, configurar en el repositorio de GitHub:
- **Secret:** `SONAR_TOKEN` — token generado en tu instancia de SonarQube
- **Variable:** `SONAR_HOST_URL` — URL de la instancia (e.g. `https://sonarcloud.io`)

Localmente:
```bash
./gradlew sonar -Dsonar.host.url=https://sonarcloud.io -Dsonar.token=TU_TOKEN
```

---

## Kubernetes

Los manifests en `k8s/` están preparados para producción:

| Archivo | Descripción |
|---|---|
| `namespace.yaml` | Namespace `customer-service` |
| `configmap.yaml` | Configuración no sensible (perfil, host DB) |
| `secret.yaml` | Credenciales de base de datos (base64) |
| `deployment.yaml` | 2 réplicas, imagen ghcr.io, probes, resource limits, securityContext |
| `service.yaml` | ClusterIP en puerto 80 |
| `ingress.yaml` | NGINX Ingress en `customer-service.local` |

Aplicar los manifests en orden:

```bash
kubectl apply -f k8s/namespace.yaml
kubectl apply -f k8s/secret.yaml
kubectl apply -f k8s/configmap.yaml
kubectl apply -f k8s/deployment.yaml
kubectl apply -f k8s/service.yaml
kubectl apply -f k8s/ingress.yaml
```

Los probes de Kubernetes apuntan a los endpoints de Actuator:
- **Liveness:** `GET /actuator/health/liveness`
- **Readiness:** `GET /actuator/health/readiness`

> Actualizar `k8s/secret.yaml` con las credenciales reales antes de desplegar en producción.  
> El Ingress asume un NGINX Ingress Controller y el host `customer-service.local`.

---

## Postman

Los archivos están en la carpeta `postman/`.

| Archivo | Propósito |
|---|---|
| `reactive-customer-service.postman_collection.json` | Colección principal con test scripts automáticos |
| `reactive-customer-service.local.postman_environment.json` | Environment local — `baseUrl = http://localhost:8080` |
| `reactive-customer-service.k8s.postman_environment.json` | Environment Kubernetes — `baseUrl = http://customer-service.local` |

---

## Autor

Proyecto orientado a demostrar habilidades en desarrollo backend reactivo con Java 21 y Spring Boot:
arquitectura hexagonal, programación reactiva con WebFlux/R2DBC, testing exhaustivo (unitario, integración, property-based, architectural), CI/CD y despliegue en Kubernetes.
