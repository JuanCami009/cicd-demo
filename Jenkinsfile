#!/usr/bin/env groovy

// ============================================================
// Pipeline CI/CD — Punto 2 del Taller
//
// Etapas:
//   1. Checkout              — obtiene el código fuente desde el SCM
//   2. Build                 — compila el proyecto con Maven
//   3. Test                  — ejecuta las pruebas unitarias
//   4. Static Analysis       — análisis estático con SonarQube + quality gate
//   5. Docker Build          — construye la imagen Docker de la aplicación
//   6. Security Scan (Trivy) — escaneo de vulnerabilidades en la imagen
//   7. Deploy                — despliega el contenedor (sólo en rama master)
//
// Prerequisitos en Jenkins (una sola vez):
//   - docker exec -u root <jenkins_id> apt-get install -y jq
//   - SonarQube corriendo: docker run -d --name sonarqube -p 9000:9000
//       -e SONAR_ES_BOOTSTRAP_CHECKS_DISABLE=true sonarqube:community
//   - Plugins: Git, Pipeline, Docker Pipeline, JUnit
// ============================================================

pipeline {

    agent any

    environment {
        APP_NAME          = 'cicd-demo'
        IMAGE_TAG         = "${APP_NAME}:latest"
        // SonarQube: host.docker.internal resuelve al host desde el contenedor Jenkins
        // en Docker Desktop (Windows/Mac). Cambiar si se usa otra red.
        SONAR_URL         = 'http://host.docker.internal:9001'
        SONAR_TOKEN       = 'squ_2ac238e5110241e98c0a2bef5c26b0e75e0f3cc4'
        SONAR_PROJECT_KEY = 'cicd-demo'
    }

    stages {

        // ----------------------------------------------------------
        // ETAPA 1: Checkout
        // ----------------------------------------------------------
        stage('Checkout') {
            steps {
                echo "==> Obteniendo código fuente desde SCM..."
                checkout scm
            }
        }

        // ----------------------------------------------------------
        // ETAPA 2: Build
        // Compila y genera el JAR. Los tests se ejecutan en la etapa
        // siguiente para separar responsabilidades.
        // ----------------------------------------------------------
        stage('Build') {
            steps {
                echo "==> Compilando aplicación con Maven..."
                sh './mvnw clean package -DskipTests'
            }
        }

        // ----------------------------------------------------------
        // ETAPA 3: Test
        // Ejecuta pruebas unitarias (sin Spring context). JaCoCo genera
        // el reporte de cobertura durante esta fase, que SonarQube
        // consumirá en la siguiente etapa.
        // ----------------------------------------------------------
        stage('Test') {
            steps {
                echo "==> Ejecutando pruebas unitarias..."
                sh './mvnw test -Dgroups=UnitTest'
            }
            post {
                always {
                    junit 'target/surefire-reports/*.xml'
                }
            }
        }

        // ----------------------------------------------------------
        // ETAPA 4: Static Analysis (SonarQube)
        //
        // Flujo:
        //   a) Verificar que SonarQube esté corriendo; si no, arrancarlo.
        //   b) Esperar que la API REST responda (health check con timeout).
        //   c) Lanzar el análisis con ./mvnw sonar:sonar.
        //   d) Leer el ceTaskId desde target/sonar/report-task.txt y hacer
        //      polling hasta que el Compute Engine termine el procesamiento.
        //   e) Consultar la API de hotspots: fallar si hay alguno TO_REVIEW.
        //
        // Quality gate: falla el pipeline si existen Security Hotspots
        // sin revisar en el proyecto.
        // ----------------------------------------------------------
        stage('Static Analysis (SonarQube)') {
            steps {
                script {

                    // a) Verificar/arrancar SonarQube
                    echo "==> Verificando que SonarQube esté disponible..."
                    def sonarRunning = sh(
                        script: "docker ps --filter 'name=sonarqube' --filter 'status=running' -q",
                        returnStdout: true
                    ).trim()

                    if (!sonarRunning) {
                        echo "==> SonarQube no está corriendo. Iniciando contenedor..."
                        sh """
                            docker start sonarqube 2>/dev/null || \
                            docker run -d \
                                --name sonarqube \
                                -p 9000:9000 \
                                -e SONAR_ES_BOOTSTRAP_CHECKS_DISABLE=true \
                                sonarqube:community
                        """
                    } else {
                        echo "==> SonarQube ya está corriendo."
                    }

                    // b) Esperar que la API responda (hasta 3 minutos)
                    echo "==> Esperando que SonarQube esté listo..."
                    timeout(time: 3, unit: 'MINUTES') {
                        waitUntil {
                            def httpCode = sh(
                                script: """
                                    curl -s -o /dev/null -w "%{http_code}" \
                                        "${SONAR_URL}/api/system/status" || echo "000"
                                """,
                                returnStdout: true
                            ).trim()
                            echo "SonarQube HTTP status: ${httpCode}"
                            return httpCode == '200'
                        }
                    }

                    // c) Lanzar análisis Maven
                    echo "==> Ejecutando análisis SonarQube..."
                    sh """
                        ./mvnw sonar:sonar \
                            -Dsonar.projectKey=${SONAR_PROJECT_KEY} \
                            -Dsonar.projectName="${APP_NAME}" \
                            -Dsonar.host.url=${SONAR_URL} \
                            -Dsonar.token=${SONAR_TOKEN} \
                            -Dsonar.coverage.jacoco.xmlReportPaths=target/site/jacoco/jacoco.xml
                    """

                    // d) Polling: esperar que el Compute Engine termine el análisis
                    echo "==> Esperando que SonarQube procese el análisis..."
                    def reportTask = readFile('target/sonar/report-task.txt')
                    def ceTaskId = ''
                    reportTask.readLines().each { line ->
                        if (line.startsWith('ceTaskId=')) {
                            ceTaskId = line.split('=', 2)[1].trim()
                        }
                    }

                    if (!ceTaskId) {
                        error "No se encontró ceTaskId en target/sonar/report-task.txt"
                    }
                    echo "==> ceTaskId: ${ceTaskId}"

                    timeout(time: 5, unit: 'MINUTES') {
                        waitUntil {
                            def taskJson = sh(
                                script: """
                                    curl -s -H "Authorization: Bearer ${SONAR_TOKEN}" \
                                        "${SONAR_URL}/api/ce/task?id=${ceTaskId}"
                                """,
                                returnStdout: true
                            ).trim()

                            def taskStatus = sh(
                                script: "echo '${taskJson}' | jq -r '.task.status'",
                                returnStdout: true
                            ).trim()

                            echo "CE task status: ${taskStatus}"

                            if (taskStatus == 'FAILED') {
                                error "El Compute Engine de SonarQube falló al procesar el análisis."
                            }

                            if (taskStatus == 'SUCCESS') {
                                return true
                            }

                            sleep(time: 10, unit: 'SECONDS')
                            return false
                        }
                    }

                    // e) Quality gate: verificar Security Hotspots sin revisar
                    echo "==> Consultando Security Hotspots..."
                    def hotspotsJson = sh(
                        script: """
                            curl -s -H "Authorization: Bearer ${SONAR_TOKEN}" \
                                "${SONAR_URL}/api/hotspots/search?projectKey=${SONAR_PROJECT_KEY}&status=TO_REVIEW"
                        """,
                        returnStdout: true
                    ).trim()

                    def hotspotCount = sh(
                        script: "echo '${hotspotsJson}' | jq '.paging.total'",
                        returnStdout: true
                    ).trim()

                    echo "==> Security Hotspots TO_REVIEW: ${hotspotCount}"

                    if (!hotspotCount || hotspotCount == 'null') {
                        error "No se pudo consultar la API de hotspots. Verifica que el token sea de tipo 'User Token' en SonarQube."
                    }

                    if (hotspotCount.toInteger() > 0) {
                        error """QUALITY GATE FAILED: SonarQube detectó ${hotspotCount} Security Hotspot(s) sin revisar.
Ver en: ${SONAR_URL}/security_hotspots?id=${SONAR_PROJECT_KEY}"""
                    }

                    echo "==> Quality Gate SonarQube: PASSED"
                }
            }
        }

        // ----------------------------------------------------------
        // ETAPA 5: Docker Build
        // ----------------------------------------------------------
        stage('Docker Build') {
            steps {
                echo "==> Construyendo imagen Docker: ${IMAGE_TAG}..."
                sh "docker build -t ${IMAGE_TAG} ."
            }
        }

        // ----------------------------------------------------------
        // ETAPA 6: Security Scan (Trivy)
        //
        // Trivy se ejecuta como contenedor Docker (sin instalación en Jenkins),
        // montando el socket Docker para acceder a imágenes locales.
        //
        // Se ejecuta dos veces:
        //   1. Reporte completo (todas las severidades) → artefacto
        //   2. Quality gate: falla si hay vulnerabilidades CRITICAL
        //
        // Flags clave:
        //   --exit-code 1      → sale con código 1 si encuentra vulns del nivel dado
        //   --severity CRITICAL → solo falla en vulnerabilidades críticas
        //   --scanners vuln    → solo vulnerabilidades (no secrets ni misconfigs)
        // ----------------------------------------------------------
        stage('Security Scan (Trivy)') {
            steps {
                script {
                    echo "==> Ejecutando escaneo de seguridad Trivy en: ${IMAGE_TAG}..."

                    // Reporte completo — no falla el pipeline, se archiva como artefacto
                    sh """
                        docker run --rm \
                            -v /var/run/docker.sock:/var/run/docker.sock \
                            -v trivy-cache:/root/.cache/trivy \
                            aquasec/trivy:latest image \
                                --exit-code 0 \
                                --severity LOW,MEDIUM,HIGH,CRITICAL \
                                --scanners vuln \
                                --timeout 10m \
                                --format table \
                                ${IMAGE_TAG} \
                            | tee trivy-full-report.txt || true
                    """

                    // Quality gate: falla si hay vulnerabilidades CRITICAL con fix disponible.
                    // --ignore-unfixed excluye CVEs sin parche publicado (no accionables).
                    def trivyExitCode = sh(
                        script: """
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
                                    ${IMAGE_TAG}
                        """,
                        returnStatus: true
                    )

                    if (trivyExitCode == 1) {
                        error """QUALITY GATE FAILED: Trivy detectó vulnerabilidades CRITICAL con fix disponible en ${IMAGE_TAG}.
Revisar el artefacto trivy-full-report.txt para el detalle completo."""
                    } else if (trivyExitCode != 0) {
                        error "Trivy finalizó con código inesperado: ${trivyExitCode}"
                    }

                    echo "==> Quality Gate Trivy: PASSED"
                }
            }
            post {
                always {
                    archiveArtifacts artifacts: 'trivy-full-report.txt', allowEmptyArchive: true
                }
            }
        }

        // ----------------------------------------------------------
        // ETAPA 7: Deploy
        // Solo se ejecuta en master y únicamente si los quality gates
        // de SonarQube y Trivy pasaron exitosamente.
        // ----------------------------------------------------------
        stage('Deploy') {
            when {
                branch 'master'
            }
            steps {
                echo "==> Desplegando contenedor ${APP_NAME} en puerto 8081..."
                sh """
                    docker stop ${APP_NAME} || true
                    docker rm   ${APP_NAME} || true
                    docker run -d \
                        --name ${APP_NAME} \
                        -p 8081:8080 \
                        --restart unless-stopped \
                        ${IMAGE_TAG}
                """
                echo "==> Aplicación disponible en http://localhost:8081"
            }
        }
    }

    post {
        always {
            echo "==> Limpiando espacio de trabajo..."
            archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
            cleanWs()
        }
        success {
            echo "==> Pipeline EXITOSO: ${APP_NAME} compilado, analizado, escaneado y desplegado."
        }
        failure {
            echo "==> Pipeline FALLIDO: revisar los logs de la etapa que falló."
        }
    }
}
