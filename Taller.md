# Ejercicio de Diseño y Construcción de Pipelines

## Prerrequisitos

* **Jenkins en Docker:**

  ```bash
  docker run -p 8080:8080 -p 50000:50000 -v jenkins_home:/var/jenkins_home jenkins/jenkins:lts
  ```

* **Configuración:**

  * Activar Jenkins
  * Instalar plugins sugeridos:

    * Git
    * Pipeline
    * Docker

* **Proyecto:**

  * Usar un repositorio local o en GitHub con código simple

* **Repositorio sugerido:**

  ```
  https://github.com/helderklemp/cicd-demogithub
  ```

---

## 1. Definición del Pipeline (30%)

### Creación del Jenkinsfile

Crear un archivo llamado `Jenkinsfile` en la raíz del proyecto.

### Etapas del Pipeline

* **Checkout:** Obtener el código fuente desde el repositorio
* **Build:** Compilar la aplicación

  * Ejemplo: `npm install`, `mvn package`
* **Docker Build:**

  ```bash
  docker build -t mi-app:latest .
  ```
* **Test:** Ejecutar pruebas básicas
* **Integración:**

  * Configurar Jenkins para usar:

    * *Pipeline script from SCM*

---

## 2. Flujo de Pipeline Avanzado (Jenkinsfile)

```groovy
pipeline {
    agent any 

    stages {

        stage('Checkout') {
            steps {
                git 'https://github.com/helderklemp/cicd-demo.git'
            }
        }

        stage('Build & Test') {
            steps {
                sh 'mvn clean package'
            }
        }

        stage('Static Analysis (SonarQube)') {
            steps {
                script {
                    sh 'mvn sonar:sonar -Dsonar.projectKey=my-app -Dsonar.host.url=http://sonarqube:9000'
                }
            }
        }

        stage('Container Security Scan (Trivy)') {
            steps {
                sh 'docker build -t mi-app:latest .'
                sh 'trivy image mi-app:latest'
            }
        }

        stage('Deploy') {
            when { branch 'main' }
            steps {
                sh 'docker run -d -p 8080:8080 mi-app:latest'
            }
        }
    }

    post {
        always {
            echo 'Limpiando entorno...'
            cleanWs()
        }
    }
}
```

---

## 3. Pasos Adicionales para el Ejercicio (40%)

### 1. Análisis Estático (20 min)

* Configurar contenedor local de **SonarQube**
* Integrar análisis en el pipeline

### 2. Escaneo de Seguridad (20 min)

* Instalar **Trivy** en Jenkins
* Añadir escaneo de vulnerabilidades

### 3. Puertas de Calidad (20 min)

* Fallar despliegue si:

  * Hay *Security Hotspots* en SonarQube
  * Trivy detecta vulnerabilidades **CRITICAL**

### 4. Limpieza e Infraestructura (30 min)

* Asegurar limpieza en `post`
* Documentar flujo en `README`

---

## 4. Despliegue y Validación (30%)

### Despliegue

```bash
docker run -d -p 80:80 mi-app:latest
```

### Refinamiento

* Manejar errores con:

```groovy
post {
    failure {
        // Notificaciones en caso de fallo
    }
}
```

### Prueba Final

1. Realizar cambios en la aplicación:

   * `index.html`
   * Código con deuda técnica o inseguro
2. Hacer **commit y push**
3. Verificar que Jenkins:

   * Detecta cambios
   * Ejecuta pipeline
   * Despliega automáticamente si pasa validaciones

---

## Resultado Esperado

Al completar el ejercicio:

* Comprenderás el ciclo completo de CI/CD
* Verás cómo la automatización:

  * Reduce intervención manual
  * Mejora la fiabilidad del software

---

## Entregables

* Export del/los jobs de Jenkins
* Capturas de configuración
* Resultados de ejecución del pipeline
* Código fuente modificado

