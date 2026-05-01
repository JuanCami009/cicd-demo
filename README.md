# cicd-demo - Taller CI/CD

Aplicación Spring Boot 2.1 usada como base para el ejercicio de diseño y construcción de pipelines CI/CD con **Jenkins** y **Kubernetes**.

---

## Tabla de contenido

- [cicd-demo - Taller CI/CD](#cicd-demo--taller-cicd)
  - [Tabla de contenido](#tabla-de-contenido)
  - [1. Descripción del proyecto](#1-descripción-del-proyecto)
  - [2. Prerrequisitos](#2-prerrequisitos)
    - [Jenkins en Docker](#jenkins-en-docker)
    - [SonarQube y herramientas de análisis](#sonarqube-y-herramientas-de-análisis)
    - [Docker Desktop con Kubernetes habilitado](#docker-desktop-con-kubernetes-habilitado)
  - [3. Arquitectura del pipeline](#3-arquitectura-del-pipeline)
  - [4. Configuración de Jenkins](#4-configuración-de-jenkins)
    - [4.1 Crear el job](#41-crear-el-job)
    - [4.2 Configurar Pipeline script from SCM](#42-configurar-pipeline-script-from-scm)
    - [4.3 Primera ejecución](#43-primera-ejecución)
    - [4.4 Segunda ejecución (pipeline con SonarQube y Trivy)](#44-segunda-ejecución-pipeline-con-sonarqube-y-trivy)
    - [4.5 Prueba final - ciclo completo](#45-prueba-final--ciclo-completo)
    - [4.6 Export del job de Jenkins](#46-export-del-job-de-jenkins)
  - [5. Ejecución local (sin Docker)](#5-ejecución-local-sin-docker)
  - [6. Ejecución con Docker](#6-ejecución-con-docker)
  - [7. Despliegue en Kubernetes (Docker Desktop)](#7-despliegue-en-kubernetes-docker-desktop)
    - [7.1 Construir la imagen (requerido antes de desplegar)](#71-construir-la-imagen-requerido-antes-de-desplegar)
    - [7.2 Aplicar los manifiestos](#72-aplicar-los-manifiestos)
    - [7.3 Verificar el despliegue](#73-verificar-el-despliegue)
    - [7.4 Acceder a la aplicación](#74-acceder-a-la-aplicación)
    - [7.5 Actualizar el despliegue](#75-actualizar-el-despliegue)
    - [7.6 Limpiar recursos de Kubernetes](#76-limpiar-recursos-de-kubernetes)
  - [8. Endpoints de la aplicación](#8-endpoints-de-la-aplicación)
  - [9. Estructura del repositorio](#9-estructura-del-repositorio)
  - [10. Variables y configuración](#10-variables-y-configuración)
    - [Variables del Jenkinsfile](#variables-del-jenkinsfile)
    - [Categorías de tests (JUnit `@Category`)](#categorías-de-tests-junit-category)
    - [Recursos Kubernetes](#recursos-kubernetes)
  - [11. Análisis de calidad y seguridad](#11-análisis-de-calidad-y-seguridad)
    - [11.1 SonarQube](#111-sonarqube)
      - [Cómo funciona el stage en el pipeline](#cómo-funciona-el-stage-en-el-pipeline)
      - [Dashboard y verificación manual](#dashboard-y-verificación-manual)
      - [Proceso de revisión de Security Hotspots](#proceso-de-revisión-de-security-hotspots)
      - [Hotspot detectado en esta aplicación](#hotspot-detectado-en-esta-aplicación)
    - [11.2 Trivy](#112-trivy)
      - [Cómo funciona el stage en el pipeline](#cómo-funciona-el-stage-en-el-pipeline-1)
      - [Vulnerabilidades encontradas en esta aplicación](#vulnerabilidades-encontradas-en-esta-aplicación)
      - [Reporte archivado en Jenkins](#reporte-archivado-en-jenkins)
      - [Ejecutar Trivy manualmente](#ejecutar-trivy-manualmente-fuera-del-pipeline)
      - [Opciones para manejar vulnerabilidades en producción](#opciones-para-manejar-vulnerabilidades-en-producción)

---

## 1. Descripción del proyecto

REST API construida con **Spring Boot 2.1** (Java 10) que expone operaciones CRUD sobre usuarios almacenados **en memoria** (sin base de datos). Sirve como proyecto de práctica para implementar un pipeline CI/CD completo.

**Stack:**

| Componente | Versión |
|---|---|
| Java | 10 (compilación) / 21 (imagen Docker - eclipse-temurin:21-jre-alpine) |
| Spring Boot | 2.1.1 |
| Maven | 3.6 (via wrapper `./mvnw`) |
| Docker | 20+ |
| Jenkins | LTS |
| Kubernetes | Docker Desktop K8s |

---

## 2. Prerrequisitos

### Jenkins en Docker

```bash
docker run -p 8080:8080 -p 50000:50000 \
  -v jenkins_home:/var/jenkins_home \
  -v /var/run/docker.sock:/var/run/docker.sock \
  jenkins/jenkins:lts
```

> **Importante:** montar `/var/run/docker.sock` le da a Jenkins acceso al daemon Docker del host, necesario para que el pipeline ejecute `docker build` y `docker run`.

**Configuración adicional del contenedor Jenkins (una sola vez):**

Instalar el Docker CLI dentro del contenedor (el socket solo es el canal, el CLI es el binario que ejecuta los comandos):

```bash
docker exec -u root <id-contenedor> apt-get update && \
docker exec -u root <id-contenedor> apt-get install -y docker.io
```

Dar permisos al socket para que el usuario `jenkins` pueda usarlo:

```bash
docker exec -u root <id-contenedor> chmod 666 /var/run/docker.sock
```

> **Nota:** el `chmod 666` se pierde si el contenedor se reinicia. Para que sea permanente, agregar esta línea al arrancar Jenkins o usar una imagen personalizada.

### SonarQube y herramientas de análisis

**Paso 1 - Instalar `jq` en el contenedor Jenkins**

`jq` es necesario para parsear las respuestas JSON de la API REST de SonarQube (polling de análisis y consulta de hotspots):

```bash
docker exec -u root <id-contenedor> apt-get update
docker exec -u root <id-contenedor> apt-get install -y jq
```

Verificar que quedó instalado:

```bash
docker exec <id-contenedor> jq --version
# jq-1.6
```

**Paso 2 - Iniciar SonarQube**

SonarQube se ejecuta como contenedor Docker en el host (no dentro de Jenkins):

```bash
docker run -d \
  --name sonarqube \
  -p 9001:9000 \
  -e SONAR_ES_BOOTSTRAP_CHECKS_DISABLE=true \
  sonarqube:community
```

> **Puerto 9001:** en Windows, el puerto 9000 puede estar ocupado por otro proceso. Se usa `-p 9001:9000` para mapear SonarQube al puerto 9001 del host. La variable `SONAR_URL` en el Jenkinsfile apunta a `http://host.docker.internal:9001` (el DNS `host.docker.internal` resuelve al host desde dentro del contenedor Jenkins).

> **Nota:** SonarQube tarda entre 60 y 90 segundos en arrancar la primera vez mientras inicializa su base de datos interna.

Verificar que está corriendo:

```bash
curl -s http://localhost:9001/api/system/status | jq '.status'
# "UP"
```

**Paso 3 - Acceder y configurar SonarQube**

1. Ir a `http://localhost:9001`
2. Login con `admin / admin` (credenciales por defecto)
3. Si el sistema solicita cambiar la contraseña, establecer la nueva y anotarla

El proyecto `cicd-demo` se crea automáticamente en SonarQube la primera vez que el pipeline ejecuta `./mvnw sonar:sonar`. No es necesario crearlo manualmente.

**Paso 4 - Generar el token de autenticación (User Token)**

El pipeline necesita un token para:
- Autenticar el análisis Maven (`./mvnw sonar:sonar`)
- Consultar la API REST de SonarQube (hotspots, estado del análisis)

> **Importante:** se debe generar un **User Token**, no un Global Analysis Token. El Global Analysis Token solo permite operaciones de escaneo, pero no tiene permisos para consultar la API de hotspots (`/api/hotspots/search`), lo que causaría un error `Insufficient privileges` en el pipeline.

Pasos para generar el token:

1. En SonarQube, ir a **My Account** (clic en el avatar, esquina superior derecha)
2. Seleccionar la pestaña **Security**
3. En la sección **Generate Tokens**:
   - **Name:** `jenkins-token` (o cualquier nombre descriptivo)
   - **Type:** `User Token`
   - **Expires in:** `No expiration` (o la política que aplique)
4. Hacer clic en **Generate**
5. **Copiar el token inmediatamente** - SonarQube no lo mostrará de nuevo

![SonarQube My Account > Security - User Token generado](sonar-qube-token.jpeg)

El token generado tiene el formato `squ_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx`.

Actualizar la variable `SONAR_TOKEN` en el `Jenkinsfile` con este valor.

**Paso 5 - Plugins de Jenkins requeridos**

Instalar desde *Manage Jenkins → Plugins → Available plugins*:

| Plugin | Uso |
|---|---|
| Git | Clonar repositorios desde SCM |
| Pipeline | Ejecutar Jenkinsfile declarativos |
| Docker Pipeline | Ejecutar comandos Docker en el pipeline |
| JUnit | Publicar reportes XML de tests en Jenkins |

> **No se requiere el plugin "SonarQube Scanner for Jenkins"**: el análisis se ejecuta directamente vía el plugin Maven de SonarQube (`sonar-maven-plugin`), lo que evita configurar una integración adicional en Jenkins y mantiene todo el control en el `Jenkinsfile`.

### Docker Desktop con Kubernetes habilitado

1. Abrir **Docker Desktop > Settings > Kubernetes**
2. Marcar **Enable Kubernetes**
3. Hacer clic en **Apply & Restart**
4. Verificar:

```bash
kubectl config current-context
# debe mostrar: docker-desktop

kubectl cluster-info
# debe mostrar la URL del cluster local
```

---

## 3. Arquitectura del pipeline

El pipeline está definido en `Jenkinsfile` con **7 etapas**:

```
┌──────────┐   ┌────────┐   ┌────────┐   ┌───────────┐   ┌──────────────┐   ┌──────────────┐   ┌────────┐
│ Checkout │──>│ Build  │──>│  Test  │──>│ SonarQube │──>│ Docker Build │──>│ Trivy  Scan  │──>│ Deploy │
│  (SCM)   │   │ (Maven)│   │(JUnit) │   │ (hotspots)│   │  (imagen)    │   │  (CRITICAL)  │   │(master)│
└──────────┘   └────────┘   └────────┘   └───────────┘   └──────────────┘   └──────────────┘   └────────┘
                                                                                                      │
                                                                                              solo rama master
```

| Etapa | Comando / Herramienta | Descripción |
|---|---|---|
| **Checkout** | `checkout scm` | Obtiene el código desde el SCM configurado en Jenkins |
| **Build** | `./mvnw clean package -DskipTests` | Compila y genera el JAR sin ejecutar tests |
| **Test** | `./mvnw test -Dgroups=UnitTest` | Ejecuta pruebas unitarias; JaCoCo genera cobertura |
| **Static Analysis** | `./mvnw sonar:sonar` + API SonarQube | Análisis estático; falla si hay Security Hotspots sin revisar |
| **Docker Build** | `docker build -t cicd-demo:latest .` | Construye la imagen Docker de la aplicación |
| **Security Scan** | `aquasec/trivy:latest image` | Escaneo de vulnerabilidades; falla si detecta CRITICAL |
| **Deploy** | `docker run -d -p 8081:8080 cicd-demo:latest` | Despliega el contenedor *(solo en rama master)* |

**Bloque `post`:**

```
always  => archiveArtifacts (JAR + trivy-full-report.txt) + cleanWs()
success => mensaje de éxito
failure => mensaje de fallo con indicación de revisar logs
```

---

## 4. Configuración de Jenkins

### 4.1 Crear el job

1. En la pantalla principal de Jenkins, hacer clic en **New Item**
2. Ingresar un nombre (ej. `cicd-demo`)
3. Seleccionar **Multibranch Pipeline** y hacer clic en **OK**

![New Item - Multibranch Pipeline](new-pipeline.jpeg)

### 4.2 Configurar Pipeline script from SCM

En la sección **Pipeline** del job:

| Campo | Valor |
|---|---|
| Definition | **Pipeline script from SCM** |
| SCM | **Git** |
| Repository URL | `https://github.com/<usuario>/cicd-demo.git` (o URL local) |
| Branch Specifier | `*/master` |
| Script Path | `Jenkinsfile` |

> Con esta configuración, Jenkins lee el `Jenkinsfile` directamente del repositorio cada vez que ejecuta el pipeline. No es necesario copiar el script en la UI.

![Pipeline from SCM - configuración](new-pipeline-config.jpeg)

### 4.3 Primera ejecución

1. Hacer clic en **Build Now**
2. En **Build History**, hacer clic en el número de build
3. Seleccionar **Console Output** para ver los logs en tiempo real

**Resultado esperado:**

```
==> Obteniendo código fuente desde SCM...
==> Compilando aplicación con Maven...
[INFO] BUILD SUCCESS
==> Ejecutando pruebas unitarias...
[INFO] Tests run: X, Failures: 0, Errors: 0
==> Construyendo imagen Docker: cicd-demo:latest...
Successfully built <image-id>
==> Pipeline EXITOSO: cicd-demo compilado, probado y desplegado.
```

![Stage View - pipeline exitoso](execution1.jpeg)

![Console Output - BUILD SUCCESS y tests](console-output-1.jpeg)

![Artifacts - JAR archivado](artifacts-1.jpeg)

### 4.4 Segunda ejecución (pipeline con SonarQube y Trivy)

Con el `Jenkinsfile` actualizado a 7 etapas, el pipeline incorpora análisis estático y escaneo de seguridad. En esta ejecución el stage **Security Scan (Trivy)** detectó vulnerabilidades CRITICAL en las dependencias de la aplicación y bloqueó el despliegue como se esperaba.

![Stage View - pipeline build #6 con Trivy fallando en Security Scan](execution2.jpeg)

![Console Output - QUALITY GATE FAILED Trivy (vulnerabilidades CRITICAL en cicd-demo:latest)](trivy-failure.jpeg)

![Artifacts del build #6 - trivy-full-report.txt disponible para descarga (175 KiB)](artifacts-2.jpeg)

### 4.5 Prueba final - ciclo completo

El objetivo es verificar que Jenkins detecta cambios en el repositorio, ejecuta el pipeline y despliega automáticamente si todas las validaciones pasan.

1. Realizar un cambio en el código fuente (ej. modificar un endpoint o agregar un comentario)
2. Hacer commit y push a la rama `master`:
   ```bash
   git add .
   git commit -m "chore: cambio para activar pipeline"
   git push origin master
   ```
3. En Jenkins, el job detecta el push automáticamente (si está configurado con webhook) o al hacer clic en **Scan Multibranch Pipeline Now**
4. Verificar que el pipeline ejecuta los 7 stages en orden
5. Si todos los quality gates pasan, la aplicación queda desplegada en `http://localhost:8081`

### 4.6 Export del job de Jenkins

El taller solicita entregar el export del job. Para obtenerlo:

1. Ir al job en Jenkins → **cicd-demo-2**
2. En el menú lateral, hacer clic en **Pipeline Syntax** o acceder directamente a la URL del config XML:
   ```
   http://localhost:8080/job/cicd-demo-2/config.xml
   ```
3. Guardar el archivo `config.xml` como parte de los entregables

Alternativamente, desde terminal:
```bash
curl -u admin:<password> http://localhost:8080/job/cicd-demo-2/config.xml -o jenkins-job-config.xml
```

---

## 5. Ejecución local (sin Docker)

```bash
# Compilar (genera target/cicd-demo-1.0-SNAPSHOT.jar)
./mvnw clean package -DskipTests

# Pruebas unitarias
./mvnw test -Dgroups=UnitTest

# Pruebas de integración (levanta Spring Boot en puerto aleatorio)
./mvnw integration-test -Dgroups=IntegrationTest

# Ejecutar la aplicación directamente
./mvnw spring-boot:run
# La app queda disponible en http://localhost:8080
```

---

## 6. Ejecución con Docker

```bash
# 1. Compilar el JAR primero
./mvnw clean package -DskipTests

# 2. Construir la imagen Docker
docker build -t cicd-demo:latest .

# 3. Ejecutar el contenedor
docker run -d \
  --name cicd-demo \
  -p 8081:8080 \
  --restart unless-stopped \
  cicd-demo:latest

# 4. Verificar que está corriendo
docker ps
curl http://localhost:8081/actuator/health

# 5. Ver logs del contenedor
docker logs cicd-demo

# 6. Detener y eliminar el contenedor
docker stop cicd-demo && docker rm cicd-demo
```

---

## 7. Despliegue en Kubernetes (Docker Desktop)

Los manifiestos se encuentran en el directorio `k8s/`:

```
k8s/
├── deployment.yml   # Deployment con 1 réplica + health probes
└── service.yml      # Service tipo NodePort (expone puerto 30080)
```

### 7.1 Construir la imagen (requerido antes de desplegar)

```bash
# La imagen debe construirse con el daemon Docker local.
# Docker Desktop comparte el daemon con el cluster K8s,
# por lo que no es necesario hacer push a ningún registry.
./mvnw clean package -DskipTests
docker build -t cicd-demo:latest .
```

### 7.2 Aplicar los manifiestos

```bash
# Crear el Deployment
kubectl apply -f k8s/deployment.yml

# Crear el Service (NodePort)
kubectl apply -f k8s/service.yml
```

### 7.3 Verificar el despliegue

![kubectl get pods - Pod Running](kubernetes-get-pods-1.jpeg)

```bash
# Ver el estado del Deployment
kubectl get deployments
# NAME        READY   UP-TO-DATE   AVAILABLE
# cicd-demo   1/1     1            1

# Ver el Pod
kubectl get pods
# NAME                         READY   STATUS    RESTARTS
# cicd-demo-<hash>             1/1     Running   0

# Ver el Service y el NodePort asignado
kubectl get svc cicd-demo-svc
# NAME            TYPE       CLUSTER-IP     PORT(S)        AGE
# cicd-demo-svc   NodePort   10.x.x.x       80:30080/TCP   Xs

# Ver logs del Pod
kubectl logs deployment/cicd-demo
```

### 7.4 Acceder a la aplicación

![Health check K8s - status UP](health-check-1.jpeg)

Con Docker Desktop K8s, el nodo es `localhost`:

```bash
# Health check
curl http://localhost:30080/actuator/health
# {"status":"UP"}

# Listar usuarios (vacío al inicio)
curl http://localhost:30080/users

# Crear un usuario
curl -X POST http://localhost:30080/users \
  -H "Content-Type: application/json" \
  -d '{"login":"juanc","email":"juan@example.com","name":"Juan Camilo"}'
```

### 7.5 Actualizar el despliegue

Cuando se construye una nueva versión de la imagen:

```bash
# Reconstruir imagen
docker build -t cicd-demo:latest .

# Forzar reinicio del Pod para que tome la nueva imagen
kubectl rollout restart deployment/cicd-demo

# Monitorear el rollout
kubectl rollout status deployment/cicd-demo
```

### 7.6 Limpiar recursos de Kubernetes

```bash
kubectl delete -f k8s/service.yml
kubectl delete -f k8s/deployment.yml
```

---

## 8. Endpoints de la aplicación

| Método | Ruta | Descripción |
|---|---|---|
| `GET` | `/` | Info del host (hostname, IP, OS) |
| `GET` | `/users` | Listar todos los usuarios |
| `GET` | `/users/{id}` | Obtener usuario por ID |
| `POST` | `/users` | Crear nuevo usuario |
| `GET` | `/actuator/health` | Health probe (usado por K8s liveness/readiness) |

**Modelo de usuario:**

```json
{
  "id": 1,
  "login": "jcmolina",
  "email": "jc@example.com",
  "name": "Juan Camilo"
}
```

> La igualdad entre usuarios se basa únicamente en `login` + `email` (no en `id`).
> Los datos se almacenan **en memoria** y se pierden al reiniciar el contenedor.

---

## 9. Estructura del repositorio

```
cicd-demo/
├── Jenkinsfile              # Pipeline CI/CD (Punto 2 del taller - 7 stages)
├── Dockerfile               # Imagen Docker: eclipse-temurin:21-jre-alpine
├── Makefile                 # Targets auxiliares para docker-compose
├── docker-compose.yml       # Servicios: builder (Maven), selenium
├── pom.xml                  # Configuración Maven / Spring Boot 2.1
├── mvnw / mvnw.cmd          # Maven wrapper (no requiere Maven instalado)
├── k8s/
│   ├── deployment.yml       # Deployment K8s (1 réplica, health probes)
│   └── service.yml          # Service NodePort (puerto 30080)
└── src/
    ├── main/java/.../
    │   ├── controller/      # ApiController, UserController
    │   ├── service/         # UserService (interface + impl en ArrayList)
    │   └── domain/          # User (builder pattern), EnvDetail
    └── test/java/.../
        ├── UserServiceTest.java       # @Category(UnitTest)
        ├── UserControllerIntTest.java # @Category(IntegrationTest)
        └── SeleniumExampleTest.java   # @Category(SystemTest)
```

---

## 10. Variables y configuración

### Variables del Jenkinsfile

| Variable | Valor por defecto | Descripción |
|---|---|---|
| `APP_NAME` | `cicd-demo` | Nombre de la aplicación y del contenedor Docker |
| `IMAGE_TAG` | `cicd-demo:latest` | Tag completo de la imagen Docker construida |
| `SONAR_URL` | `http://host.docker.internal:9001` | URL de SonarQube accesible desde el contenedor Jenkins |
| `SONAR_TOKEN` | `squ_...` | User Token generado en SonarQube (My Account → Security → Type: User Token) |
| `SONAR_PROJECT_KEY` | `cicd-demo` | Identificador del proyecto en SonarQube |

### Categorías de tests (JUnit `@Category`)

| Categoría | Comando | Descripción |
|---|---|---|
| `UnitTest` | `./mvnw test -Dgroups=UnitTest` | Tests sin Spring context (rápidos) |
| `IntegrationTest` | `./mvnw integration-test -Dgroups=IntegrationTest` | Tests HTTP con `@SpringBootTest` |
| `SystemTest` | `APP_URL=http://... ./mvnw test -Dgroups=SystemTest` | Tests Selenium contra app desplegada |

### Recursos Kubernetes

| Recurso | Tipo | Puerto externo | Puerto interno |
|---|---|---|---|
| `cicd-demo` | Deployment | - | 8080 |
| `cicd-demo-svc` | Service (NodePort) | 30080 | 8080 |

---

## 11. Análisis de calidad y seguridad

### 11.1 SonarQube

SonarQube analiza el código fuente en busca de bugs, code smells y **Security Hotspots**. El pipeline falla si detecta algún hotspot en estado `TO_REVIEW`, bloqueando el despliegue hasta que sea revisado y marcado como resuelto o aceptado.

#### Cómo funciona el stage en el pipeline

El stage `Static Analysis (SonarQube)` ejecuta el siguiente flujo:

1. **Verificar que SonarQube esté corriendo** - si el contenedor no está activo, intenta iniciarlo con `docker start sonarqube`
2. **Health check** - hace polling a `/api/system/status` con timeout de 3 minutos hasta que responda HTTP 200
3. **Lanzar análisis Maven:**
   ```bash
   ./mvnw sonar:sonar \
     -Dsonar.projectKey=cicd-demo \
     -Dsonar.host.url=http://host.docker.internal:9001 \
     -Dsonar.token=<USER_TOKEN> \
     -Dsonar.coverage.jacoco.xmlReportPaths=target/site/jacoco/jacoco.xml
   ```
4. **Polling del Compute Engine** - lee el `ceTaskId` de `target/sonar/report-task.txt` y consulta `/api/ce/task?id=<ceTaskId>` cada 10 segundos hasta que el estado sea `SUCCESS` (timeout 5 minutos)
5. **Quality gate - Security Hotspots** - consulta `/api/hotspots/search?projectKey=cicd-demo&status=TO_REVIEW`; si `paging.total > 0`, el pipeline falla con `error()`

> **Por qué se usa la API de hotspots y no el quality gate nativo de SonarQube:** la API `/api/qualitygates/project_status` no incluye Security Hotspots por defecto sin configuración adicional en SonarQube. La API `/api/hotspots/search` es directa y no requiere configuración extra del quality gate.

![Jenkins Console Output - QUALITY GATE FAILED SonarQube (1 Security Hotspot sin revisar)](sonar-qube-failure.jpeg)

#### Dashboard y verificación manual

**Acceder al dashboard del proyecto:**

```
http://localhost:9001/dashboard?id=cicd-demo
```

**Verificar Security Hotspots via API REST:**

```bash
# Con token de usuario (mismo que usa el pipeline)
curl -s \
  -H "Authorization: Bearer squ_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx" \
  "http://localhost:9001/api/hotspots/search?projectKey=cicd-demo&status=TO_REVIEW" \
  | jq '.paging.total'
# Resultado esperado cuando no hay hotspots pendientes: 0
```

**Ver hotspots pendientes en la UI:**

```
http://localhost:9001/security_hotspots?id=cicd-demo
```

![SonarQube Security Hotspots - hotspot CSRF detectado en ApiController.java (Status: To review)](sonar-qube-failure-panel.jpeg)

#### Proceso de revisión de Security Hotspots

Cuando el pipeline falla por hotspots detectados, el flujo de revisión es:

1. Ir a `http://localhost:9001/security_hotspots?id=cicd-demo`
2. Seleccionar el hotspot de la lista
3. Revisar el código afectado, la descripción del riesgo y las recomendaciones de SonarQube
4. Hacer clic en **Review** y seleccionar el estado correspondiente:
   - **Safe** - el código fue revisado y no representa un riesgo real en este contexto
   - **Fixed** - se corrigió el código para eliminar el riesgo
   - **Acknowledged** - el riesgo es conocido y aceptado (requiere justificación)
5. Guardar el cambio
6. Volver a ejecutar el pipeline - ahora el hotspot ya no aparece en `TO_REVIEW` y el quality gate pasa

#### Hotspot detectado en esta aplicación

Durante el análisis, SonarQube detectó **1 Security Hotspot** en `ApiController.java` (línea 13):

- **Regla:** `java:S3752` - Make sure allowing safe and unsafe HTTP methods is safe here
- **Descripción:** el endpoint `GET /` no restringe explícitamente los métodos HTTP permitidos, lo que podría exponer la ruta a peticiones con métodos no seguros (CSRF)
- **Resolución:** marcado como **Safe** - la ruta solo devuelve información del entorno y no modifica estado del servidor, por lo que no representa un riesgo de CSRF real

---

### 11.2 Trivy

Trivy escanea la imagen Docker en busca de vulnerabilidades conocidas (CVEs) en paquetes del sistema operativo y dependencias de la aplicación. El pipeline falla si detecta alguna vulnerabilidad **CRITICAL**, bloqueando el despliegue.

Trivy se ejecuta como contenedor Docker (no requiere instalación en Jenkins), accediendo al daemon Docker del host a través del socket montado.

#### Cómo funciona el stage en el pipeline

El stage `Security Scan (Trivy)` ejecuta Trivy **dos veces**:

**Primera ejecución - reporte completo (no falla el pipeline):**
```bash
docker run --rm \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v trivy-cache:/root/.cache/trivy \
  aquasec/trivy:latest image \
    --exit-code 0 \
    --severity LOW,MEDIUM,HIGH,CRITICAL \
    --scanners vuln \
    --timeout 10m \
    --format table \
    cicd-demo:latest | tee trivy-full-report.txt
```

**Segunda ejecución - quality gate (falla si hay CRITICAL con fix disponible):**
```bash
docker run --rm \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v trivy-cache:/root/.cache/trivy \
  aquasec/trivy:latest image \
    --exit-code 1 \
    --severity CRITICAL \
    --ignore-unfixed \
    --scanners vuln \
    --timeout 10m \
    --quiet \
    cicd-demo:latest
```

Flags clave:
- `--exit-code 1` - Trivy sale con código 1 si encuentra al menos 1 vulnerabilidad del nivel indicado
- `--ignore-unfixed` - excluye CVEs sin parche publicado; el pipeline solo falla en vulnerabilidades accionables (para las que existe una versión corregida del paquete)
- `--scanners vuln` - solo escanea vulnerabilidades (excluye secrets y misconfigurations)
- `-v trivy-cache:/root/.cache/trivy` - volumen Docker que persiste la base de datos de CVEs entre builds (evita descargarla en cada ejecución)
- El patrón sibling container con `/var/run/docker.sock` permite a Trivy acceder a las imágenes locales del daemon Docker del host

#### Vulnerabilidades encontradas en esta aplicación

Trivy detectó **22 vulnerabilidades CRITICAL** en `app.jar` provenientes de las dependencias de Spring Boot 2.1.1 (versión de 2018). Las principales son:

| Paquete | Versión | CVEs críticos (ejemplos) |
|---|---|---|
| `jackson-databind` | 2.9.7 | CVE-2019-14379, CVE-2019-14540, CVE-2019-16335, CVE-2020-8840 |
| `tomcat-embed-core` | 9.0.13 | CVE-2019-0199, CVE-2019-10072, CVE-2020-1938 |
| `spring-boot-actuator` | 2.1.1 | CVE-2022-22965 (Spring4Shell) |
| `spring-webmvc` | 5.1.3 | CVE-2022-22965 (Spring4Shell) |
| `spring-beans` | 5.1.3 | CVE-2022-22965 (Spring4Shell) |

Estas vulnerabilidades existen porque la aplicación usa Spring Boot 2.1.1, una versión desactualizada sin parches de seguridad. El quality gate funcionó correctamente al detectarlas y bloquear el despliegue.

#### Reporte archivado en Jenkins

Cada ejecución genera el artefacto `trivy-full-report.txt` con el reporte completo (todas las severidades) disponible para descarga desde *Build → Artifacts* en la UI de Jenkins.

#### Ejecutar Trivy manualmente (fuera del pipeline)

```bash
# Reporte completo con todas las severidades
docker run --rm \
  -v /var/run/docker.sock:/var/run/docker.sock \
  aquasec/trivy:latest image \
    --severity LOW,MEDIUM,HIGH,CRITICAL \
    cicd-demo:latest

# Solo CRITICAL (simula el quality gate del pipeline)
docker run --rm \
  -v /var/run/docker.sock:/var/run/docker.sock \
  aquasec/trivy:latest image \
    --exit-code 1 \
    --severity CRITICAL \
    cicd-demo:latest
echo "Exit code: $?"
# 0 = ninguna CRITICAL; 1 = al menos 1 CRITICAL encontrada
```

#### Opciones para manejar vulnerabilidades en producción

En un proyecto real, existen varias estrategias para manejar el quality gate de Trivy:

| Estrategia | Comando adicional | Cuándo usar |
|---|---|---|
| Solo fallar en CVEs con fix disponible | `--ignore-unfixed` | Cuando hay CVEs sin parche publicado aún |
| Ignorar CVEs específicos | Archivo `.trivyignore` con lista de CVE IDs | CVEs revisados y aceptados formalmente |
| Actualizar dependencias | Actualizar `pom.xml` a versiones con parche | Solución definitiva recomendada |
