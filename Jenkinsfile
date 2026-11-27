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
                .venv/bin/python - << 'EOF'
import os, time, urllib.request, urllib.parse

os.makedirs("reports", exist_ok=True)
key_path = "/run/secrets/zap_api_key"
api_key = open(key_path).read().strip() if os.path.exists(key_path) else os.environ.get("ZAP_API_KEY", "")
target = urllib.parse.quote("http://ev3-jenkins:5001/", safe="")

def zap_get(path):
    url = f"http://ev3-zap:8080{path}"
    req = urllib.request.Request(url, headers={"X-ZAP-API-Key": api_key})
    with urllib.request.urlopen(req, timeout=30) as r:
        return r.read().decode("utf-8", errors="ignore")

print("1. Ejecutando Spider OWASP ZAP...")
print(zap_get(f"/JSON/spider/action/scan/?url={target}"))
time.sleep(6)

print("2. Ejecutando Active Scan OWASP ZAP...")
print(zap_get(f"/JSON/ascan/action/scan/?url={target}"))
time.sleep(8)

print("3. Descargando reporte HTML...")
html = zap_get("/OTHER/core/other/htmlreport/")
with open("reports/reporte-zap.html", "w", encoding="utf-8") as f:
    f.write(html)
print(f"Reporte generado correctamente: {len(html)} bytes")
EOF
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
            // Registro de trazabilidad: build <-> versión <-> commit <-> pruebas <-> hall