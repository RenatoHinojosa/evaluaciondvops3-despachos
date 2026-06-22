# Backend Despachos (rama deploy)

Microservicio Java/Spring Boot para la gestión de despachos.

## 1) Arquitectura backend

Estructura principal del código:

- **Entrada HTTP:** `src/main/java/com/citt/controller/DespachoController.java`
- **Lógica de negocio:** `src/main/java/com/citt/persistence/services/DespachoService.java` y `DespachoServiceImpl.java`
- **Acceso a datos:** `src/main/java/com/citt/persistence/repository/DespachoRepository.java`
- **Entidad JPA:** `src/main/java/com/citt/persistence/entity/Despacho.java`
- **Manejo de errores:** `src/main/java/com/citt/exceptions/*`
- **Configuración OpenAPI/CORS:** `src/main/java/com/citt/config/*`

Stack definido en `pom.xml`:

- Java 17
- Spring Boot 3.4.4
- Spring Web
- Spring Data JPA + Hibernate
- MySQL Connector/J
- springdoc-openapi (Swagger UI)
- Lombok

## 2) Configuración de MySQL por variables de entorno

Configuración en:
`src/main/resources/application.properties`

Variables requeridas:

- `DB_ENDPOINT`
- `DB_PORT`
- `DB_NAME`
- `DB_USERNAME`
- `DB_PASSWORD`

La URL JDBC se construye así:

`jdbc:mysql://${DB_ENDPOINT}:${DB_PORT}/${DB_NAME}?useSSL=false&serverTimezone=UTC&createDatabaseIfNotExist=true&allowPublicKeyRetrieval=true`

Además:

- Puerto de la app: `server.port=8081`
- Swagger UI: `/swagger-ui.html`

## 3) Flujo CI/CD real en deploy

Workflow principal:
`.github/workflows/main.yml`

Se dispara en push a la rama `deploy` y tiene 2 jobs:

1. **build-and-push**
   - Checkout
   - Configura credenciales AWS
   - Login en ECR
   - `docker build` de la imagen
   - Push de tags `${github.sha}` y `latest` a ECR

2. **deploy-to-ec2**
   - Configura credenciales AWS
   - Ejecuta `aws ssm send-command` sobre una instancia EC2
   - En EC2:
     - Login a ECR
     - Pull de imagen `latest`
     - Crea red Docker
     - Detiene/elimina contenedores previos
     - Levanta MySQL (`mysql:8.0`) con variables `MYSQL_*`
     - Levanta backend con variables `DB_*`

> **Importante:** En esta rama, el deploy automatizado está implementado para **EC2 + Docker vía SSM**, no para EKS.

## 4) Kubernetes/EKS: recursos existentes y faltantes

Recursos existentes en `k8s/`:

- `deployment.yaml` (backend)
- `service.yaml` (backend, ClusterIP)
- `mysql-deployment.yaml`
- `mysql-service-yaml.yaml`

Observaciones clave:

- `deployment.yaml` de backend usa imagen placeholder (`nginx:alpine`) con comentario de reemplazo por workflow.
- Se referencia un secreto `despacho-db-secret`, pero no existe manifiesto de `Secret` en el repo.
- MySQL usa `emptyDir` (almacenamiento efímero).
- No hay `Ingress`, `PVC`, `StorageClass`, `HPA`, `Namespace` ni chart Helm.
- No existe workflow de GitHub Actions para deploy a EKS (`kubectl`/`helm`).
- Hay desalineación de puertos: app escucha en **8081**, pero `k8s/deployment.yaml` declara `containerPort: 8080`.

## 5) Endpoints y documentación

Prefijo base:

- `/api/v1/despachos`

Endpoints:

- `GET /api/v1/despachos` → listar
- `GET /api/v1/despachos/{idDespacho}` → obtener por ID
- `POST /api/v1/despachos` → crear
- `PUT /api/v1/despachos/{idDespacho}` → actualizar
- `DELETE /api/v1/despachos/{idDespacho}` → eliminar

Documentación interactiva:

- Swagger UI: `http://localhost:8081/swagger-ui.html`

## 6) Guía para clonar, configurar y ejecutar localmente

### Requisitos

- Java 17
- Maven (o wrapper `./mvnw`)
- Docker (opcional, para ejecución containerizada)
- MySQL accesible

### Clonar

```bash
git clone https://github.com/RenatoHinojosa/evaluaciondvops3-despachos.git
cd evaluaciondvops3-despachos
```

### Variables de entorno (ejemplo)

```bash
export DB_ENDPOINT=localhost
export DB_PORT=3306
export DB_NAME=despachos_db
export DB_USERNAME=root
export DB_PASSWORD=tu_clave
```

### Ejecutar con Maven

```bash
chmod +x mvnw
./mvnw spring-boot:run
```

### Ejecutar con Docker

```bash
docker build -t despachos-app:local .
docker run --rm -p 8081:8081 \
  -e DB_ENDPOINT=host.docker.internal \
  -e DB_PORT=3306 \
  -e DB_NAME=despachos_db \
  -e DB_USERNAME=root \
  -e DB_PASSWORD=tu_clave \
  despachos-app:local
```

## Despliegue en EKS (estado actual y qué falta)

Hoy el repo **no tiene pipeline completo EKS**. Para dejarlo operativo en EKS se debe completar al menos:

1. Crear/ajustar manifiesto `Secret` (`despacho-db-secret`).
2. Corregir puertos en Deployment/Service para 8081.
3. Reemplazar `emptyDir` por `PersistentVolumeClaim` para MySQL.
4. Agregar exposición externa (`Ingress` o `Service type LoadBalancer`).
5. Incorporar workflow GitHub Actions para autenticación AWS + actualización de kubeconfig + `kubectl apply`/`helm upgrade`.
6. Definir estrategia de imagen en Deployment (tag inmutable por SHA).

## Notas de validación

- `./mvnw test` actualmente falla en entorno limpio si no se definen `DB_ENDPOINT` y `DB_PORT` (el test de contexto intenta inicializar datasource MySQL).
