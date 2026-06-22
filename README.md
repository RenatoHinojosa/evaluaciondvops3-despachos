# Backend Despachos (rama `deploy`)

Este README describe **cómo funciona la rama `deploy`** del backend de despachos: arquitectura Java/Spring Boot, base de datos MySQL por variables de entorno, pipeline CI/CD a AWS (ECR + EKS), endpoints y pasos de ejecución/despliegue.

## 1) Arquitectura del backend (Spring Boot)

Proyecto Maven (`pom.xml`) con Java 17 y Spring Boot 3.4.x.

Capas principales:

- **API REST**: `src/main/java/com/citt/controller/DespachoController.java`
  - Base path: `api/v1/despachos`
  - Endpoints CRUD para despachos.
- **Servicio**:
  - Interfaz: `.../persistence/services/DespachoService.java`
  - Implementación: `.../persistence/services/DespachoServiceImpl.java`
  - Contiene la lógica de negocio y validaciones de existencia.
- **Persistencia**:
  - Entidad JPA: `.../persistence/entity/Despacho.java`
  - Repositorio: `.../persistence/repository/DespachoRepository.java` (`JpaRepository`).
- **Manejo de errores**:
  - `.../exceptions/RestResponseEntityExceptionHandler.java`
  - Maneja `DespachoNotFoundException` y errores de validación.
- **Configuración transversal**:
  - CORS abierto (`CorsConfig`).
  - OpenAPI/Swagger (`OpenApiConfig`).

Dependencias relevantes (`pom.xml`):
- `spring-boot-starter-web`
- `spring-boot-starter-data-jpa`
- `mysql-connector-j`
- `springdoc-openapi-starter-webmvc-ui`
- `spring-boot-starter-validation`

## 2) Configuración MySQL por variables de entorno

En `src/main/resources/application.properties`:

- `spring.datasource.url=jdbc:mysql://${DB_ENDPOINT}:${DB_PORT}/${DB_NAME}...`
- `spring.datasource.username=${DB_USERNAME}`
- `spring.datasource.password=${DB_PASSWORD}`
- `server.port=8081`
- `spring.jpa.hibernate.ddl-auto=update`

Variables requeridas para la app:

- `DB_ENDPOINT`
- `DB_PORT`
- `DB_NAME`
- `DB_USERNAME`
- `DB_PASSWORD`

### Cómo se inyectan en EKS

En `.github/workflows/main.yml`, el job de deploy crea/actualiza el secret `despacho-db-secret` en Kubernetes con:

- `MYSQL_DATABASE` ← `secrets.DB_NAME`
- `MYSQL_ROOT_PASSWORD` ← `secrets.DB_PASSWORD`
- `MYSQL_USER` ← `secrets.DB_USER`
- `MYSQL_PASSWORD` ← `secrets.DB_PASSWORD`

Luego, los manifests K8s consumen ese secret:

- `k8s/deployment.yaml` mapea a variables `DB_*` del contenedor backend.
- `k8s/mysql-deployment.yaml` mapea a variables `MYSQL_*` del contenedor MySQL.

## 3) Flujo CI/CD exacto en `deploy` (GitHub Actions → ECR → EKS)

Archivo: `.github/workflows/main.yml`

Trigger:
- `push` a rama `deploy`
- `workflow_dispatch` manual

### Job 1: `build-and-push`
1. Checkout.
2. Configura credenciales AWS.
3. Login a ECR.
4. Define tag de imagen = `github.sha`.
5. `docker build` con dos tags:
   - `${sha}`
   - `latest`
6. Push de ambas tags a ECR.

### Job 2: `deploy-to-eks` (depende de job 1)
1. Checkout.
2. Configura credenciales AWS.
3. Instala `kubectl` (v1.29.0).
4. `aws eks update-kubeconfig` para el cluster objetivo.
5. Crea/aplica secret `despacho-db-secret` en namespace de destino.
6. `kubectl apply -f k8s/ -n <namespace>`.
7. `kubectl set image` sobre deployment backend para fijar la imagen nueva desde ECR.
8. Espera `kubectl rollout status`.
9. Muestra pods y services.

### Secrets que usa el workflow

- `AWS_ACCOUNT_ID`
- `AWS_REGION`
- `AWS_ACCESS_KEY_ID`
- `AWS_SECRET_ACCESS_KEY`
- `AWS_SESSION_TOKEN`
- `AWS_ECR_REPOSITORY`
- `EKS_CLUSTER_NAME`
- `EKS_NAMESPACE`
- `K8S_DEPLOYMENT_NAME` (actualmente `despacho`)
- `K8S_CONTAINER_NAME` (actualmente `despacho-app`)
- `DB_NAME`
- `DB_USER`
- `DB_PASSWORD`

## 4) Endpoints y documentación expuesta

Base path API: `/api/v1/despachos`

- `POST /api/v1/despachos`
- `PUT /api/v1/despachos/{idDespacho}`
- `GET /api/v1/despachos`
- `GET /api/v1/despachos/{idDespacho}`
- `DELETE /api/v1/despachos/{idDespacho}`

Documentación:
- Swagger UI: `/swagger-ui.html`
- OpenAPI JSON (por defecto springdoc): `/v3/api-docs`

## 5) Cómo clonar, configurar, ejecutar local y desplegar en EKS

## Clonado y rama correcta

```bash
git clone https://github.com/RenatoHinojosa/evaluaciondvops3-despachos.git
cd evaluaciondvops3-despachos
git checkout deploy
```

## Ejecución local (sin Docker)

1. Levanta un MySQL accesible localmente (ejemplo: `localhost:3306`).
2. Exporta variables:

```bash
export DB_ENDPOINT=localhost
export DB_PORT=3306
export DB_NAME=despachos
export DB_USERNAME=root
export DB_PASSWORD=tu_password
```

3. Ejecuta:

```bash
mvn clean spring-boot:run
```

API local: `http://localhost:8081`  
Swagger: `http://localhost:8081/swagger-ui.html`

## Ejecución con Docker

```bash
docker build -t despacho-backend:local .
docker run --rm -p 8081:8081 \
  -e DB_ENDPOINT=host.docker.internal \
  -e DB_PORT=3306 \
  -e DB_NAME=despachos \
  -e DB_USERNAME=root \
  -e DB_PASSWORD=tu_password \
  despacho-backend:local
```

## Manifiestos Kubernetes incluidos (`k8s/`)

- `deployment.yaml`: backend Spring Boot.
- `service.yaml`: Service ClusterIP del backend (`despacho-service`, puerto 8081).
- `mysql-deployment.yaml`: MySQL 8.0 con `emptyDir` (datos efímeros).
- `mysql-service.yaml`: Service ClusterIP de MySQL (`despacho-mysql-service`, puerto 3306).

## Despliegue en EKS (vía pipeline recomendado)

1. Tener repositorio ECR creado y cluster EKS operativo.
2. Configurar todos los secrets del workflow.
3. Hacer push a `deploy` (o lanzar `workflow_dispatch`).
4. Verificar en Actions que:
   - se publicó imagen en ECR,
   - rollout de `despacho` terminó OK en namespace configurado.

## Notas importantes de operación

- El pipeline reemplaza la imagen del deployment con `kubectl set image`, por eso en `k8s/deployment.yaml` hay una imagen placeholder (`nginx:alpine`).
- `mysql-deployment.yaml` usa `emptyDir`; si el pod reinicia se pierde data. Para producción se recomienda PVC.
- El backend escucha en `8081` (`application.properties`); mantener puertos coherentes al ajustar manifests/servicios.
- En `k8s/deployment.yaml` el `containerPort` figura como `8080`, mientras la app y el Service usan `8081`; el tráfico funciona por `targetPort: 8081`, pero conviene mantener esta definición consistente.
