// Jenkinsfile - Pipeline de "Sistema Alumnos" (SecureDev · Evaluación Parcial 3 · OCY1102)
pipeline {
    agent any

    options {
        timestamps()
        timeout(time: 40, unit: 'MINUTES')
        disableConcurrentBuilds()
    }

    // Revisa GitHub cada ~3 minutos: cada push o PR de Dependabot fusionado dispara un build.
    triggers {
        pollSCM('H/3 * * * *')
    }

    environment {
        PIP_DISABLE_PIP_VERSION_CHECK = '1'
        SEGURIDAD_OK = 'true'
    }

    stages {
        stage('Build') {
            steps {
                script {
                    // Versión = 1.0.<nº de build>+<commit corto>: une cada despliegue con su commit (trazabilidad)
                    def sha = sh(returnStdout: true, script: 'git rev-parse --short HEAD').trim()
                    env.APP_VERSION = "1.0.${env.BUILD_NUMBER}+${sha}"
                }
                echo "Versión de este build: ${env.APP_VERSION}"
                sh 'git log -1 --format="Commit %h de %an: %s"'
                // Cada build parte limpio: sin reportes viejos y con un entorno virtual nuevo
                sh 'rm -rf .venv reports && mkdir -p reports'
                sh 'python3 -m venv .venv'
                sh '.venv/bin/pip install --quiet -r requirements.txt -r requirements-dev.txt'
                sh '.venv/bin/python -m compileall -q app.py'
            }
        }

        stage('Test') {
            steps {
                sh '.venv/bin/python -m pytest --junitxml=reports/junit.xml'
            }
            post {
                always {
                    junit allowEmptyResults: true, testResults: 'reports/junit.xml'
                }
            }
        }

        stage('Security Review (SAST)') {
            steps {
                script {
                    def statusBandit = sh(
                        script: '.venv/bin/python -m bandit -r app.py -f json -o reports/bandit.json && .venv/bin/python -m bandit -r app.py -f txt -o reports/bandit.txt',
                        returnStatus: true
                    )
                    sh 'cat reports/bandit.txt || true'
                    if (statusBandit != 0) {
                        env.SEGURIDAD_OK = 'false'
                        echo '[ALERTA SAST] Bandit encontró vulnerabilidades en el código.'
                    }
                }
            }
        }

        stage('Dependency Management') {
            steps {
                script {
                    def statusAudit = sh(
                        script: '.venv/bin/python -m pip_audit -r requirements.txt --format json -o reports/pip-audit.json && .venv/bin/python -m pip_audit -r requirements.txt > reports/pip-audit.txt',
                        returnStatus: true
                    )
                    sh 'cat reports/pip-audit.txt || true'
                    if (statusAudit != 0) {
                        env.SEGURIDAD_OK = 'false'
                        echo '[ALERTA SCA] pip-audit encontró dependencias con vulnerabilidades conocidas.'
                    }
                }
            }
        }

        stage('Deploy Staging') {
            steps {
                sh 'bash scripts/deploy.sh staging 5001'
                sh 'sleep 3 && curl -f http://localhost:5001/health'
            }
        }

        stage('DAST - OWASP ZAP') {
            steps {
                sh '''
                mkdir -p reports
                .venv/bin/python -c 'import os, time, urllib.request, urllib.parse; api_key = open("/run/secrets/zap_api_key").read().strip() if os.path.exists("/run/secrets/zap_api_key") else ""; target = urllib.parse.quote("http://ev3-jenkins:5001/", safe=""); req = lambda p: urllib.request.urlopen(urllib.request.Request(f"http://ev3-zap:8080{p}", headers={"X-ZAP-API-Key": api_key}), timeout=30).read().decode("utf-8", "ignore"); print("Spider:", req(f"/JSON/spider/action/scan/?url={target}")); time.sleep(6); print("Scan:", req(f"/JSON/ascan/action/scan/?url={target}")); time.sleep(8); html = req("/OTHER/core/other/htmlreport/"); open("reports/reporte-zap.html", "w", encoding="utf-8").write(html); print("OK:", len(html), "bytes")'
                ls -lh reports/reporte-zap.html
                '''
            }
        }

        stage('Deploy Producción') {
            when {
                expression { env.SEGURIDAD_OK == 'true' }
            }
            steps {
                sh 'bash scripts/deploy.sh produccion 5000'
                script { env.PRODUCCION = env.APP_VERSION }
            }
        }
    }

    post {
        always {
            // Registro de trazabilidad: build <-> versión <-> commit <-> pruebas <-> hallazgos <-> despliegue
            withEnv(["RESULTADO=${currentBuild.currentResult}"]) {
                sh 'python3 scripts/trazabilidad.py'
            }
            archiveArtifacts artifacts: 'reports/**', allowEmptyArchive: true
        }
    }
}