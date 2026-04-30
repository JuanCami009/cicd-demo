#!/usr/bin/env groovy

// ============================================================
// Pipeline CI/CD — Punto 1 del Taller
//
// Etapas:
//   1. Checkout    — obtiene el código fuente desde el SCM
//   2. Build       — compila el proyecto con Maven
//   3. Test        — ejecuta las pruebas unitarias
//   4. Docker Build— construye la imagen Docker de la aplicación
//   5. Deploy      — despliega el contenedor (sólo en rama master)
//
// Prerequisito en Jenkins:
//   - Job configurado como "Pipeline script from SCM"
//   - Plugins: Git, Pipeline, Docker Pipeline
// ============================================================

pipeline {

    // Ejecuta en cualquier agente disponible del nodo Jenkins
    agent any

    environment {
        APP_NAME  = 'cicd-demo'
        // Tag de la imagen Docker que se construye y despliega
        IMAGE_TAG = "${APP_NAME}:latest"
    }

    stages {

        // ----------------------------------------------------------
        // ETAPA 1: Checkout
        // Descarga el código fuente desde el repositorio configurado
        // en la sección "Pipeline script from SCM" del job de Jenkins.
        // 'checkout scm' usa automáticamente la URL y branch del job,
        // por lo que no es necesario hardcodear ninguna URL.
        // ----------------------------------------------------------
        stage('Checkout') {
            steps {
                echo "==> Obteniendo código fuente desde SCM..."
                checkout scm
            }
        }

        // ----------------------------------------------------------
        // ETAPA 2: Build
        // Compila la aplicación y genera el artefacto JAR.
        // Se omiten los tests aquí (-DskipTests) porque la etapa
        // siguiente los ejecuta de forma dedicada, manteniendo
        // separadas las responsabilidades de compilación y prueba.
        // ----------------------------------------------------------
        stage('Build') {
            steps {
                echo "==> Compilando aplicación con Maven..."
                sh './mvnw clean package -DskipTests'
            }
        }

        // ----------------------------------------------------------
        // ETAPA 3: Test
        // Ejecuta únicamente las pruebas unitarias (categoría UnitTest).
        // Estas pruebas no levantan Spring context ni base de datos,
        // por lo que son rápidas y adecuadas para validación rápida.
        // Los resultados XML se publican en Jenkins (JUnit report).
        // ----------------------------------------------------------
        stage('Test') {
            steps {
                echo "==> Ejecutando pruebas unitarias..."
                sh './mvnw test -Dgroups=UnitTest'
            }
            post {
                // Publica el reporte de tests en Jenkins siempre,
                // incluso si alguna prueba falla, para tener visibilidad.
                always {
                    junit 'target/surefire-reports/*.xml'
                }
            }
        }

        // ----------------------------------------------------------
        // ETAPA 4: Docker Build
        // Construye la imagen Docker usando el Dockerfile del proyecto.
        // La imagen se etiqueta como cicd-demo:latest y queda disponible
        // en el daemon Docker local del agente Jenkins.
        // Requiere que el JAR ya esté generado (etapa Build anterior).
        // ----------------------------------------------------------
        stage('Docker Build') {
            steps {
                echo "==> Construyendo imagen Docker: ${IMAGE_TAG}..."
                sh "docker build -t ${IMAGE_TAG} ."
            }
        }

        // ----------------------------------------------------------
        // ETAPA 5: Deploy
        // Despliega la aplicación como contenedor Docker en el agente.
        // Solo se ejecuta en la rama 'master' (producción/integración).
        // Detiene y elimina el contenedor anterior si existe, evitando
        // conflictos de nombre y de puerto entre ejecuciones del pipeline.
        // ----------------------------------------------------------
        stage('Deploy') {
            when {
                branch 'master'
            }
            steps {
                echo "==> Desplegando contenedor ${APP_NAME} en puerto 8080..."
                sh """
                    docker stop ${APP_NAME} || true
                    docker rm   ${APP_NAME} || true
                    docker run -d \
                        --name ${APP_NAME} \
                        -p 8080:8080 \
                        --restart unless-stopped \
                        ${IMAGE_TAG}
                """
                echo "==> Aplicación disponible en http://localhost:8080"
            }
        }
    }

    // --------------------------------------------------------------
    // POST: Acciones que se ejecutan al finalizar el pipeline,
    // independientemente del resultado de las etapas.
    // --------------------------------------------------------------
    post {
        always {
            echo "==> Limpiando espacio de trabajo..."
            // Preserva el JAR como artefacto descargable desde Jenkins UI
            archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
            // Elimina el workspace para no acumular espacio en el agente
            cleanWs()
        }
        success {
            echo "==> Pipeline EXITOSO: ${APP_NAME} compilado, probado y desplegado."
        }
        failure {
            echo "==> Pipeline FALLIDO: revisar los logs de la etapa que falló."
        }
    }
}
