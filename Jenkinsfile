pipeline {
    agent any

    environment {
        DEFECTDOJO_URL = 'http://host.docker.internal:8082'
        DEFECTDOJO_API_KEY = credentials('defectdojo_api_token')
        DEFECTDOJO_PRODUCT_NAME = 'OWASP Juice Shop Pipeline'
    }

    stages {

        stage('Build') {
            steps {
                echo "⚙️ Installing dependencies for Juice Shop (Node)…"
                sh '''
                docker run --rm -v $(pwd):/app -w /app node:18 bash -c "npm install"
                '''
            }
        }

        stage('SAST - Semgrep') {
            steps {
                echo "🔍 Running Semgrep SAST scan…"
                sh '''
                docker run --rm -v $(pwd):/src -w /src semgrep/semgrep semgrep \
                  --config=auto \
                  --json \
                  --output semgrep-report.json || true
                '''
                archiveArtifacts artifacts: 'semgrep-report.json', fingerprint: true
            }
        }

        stage('SCA - Trivy (File System)') {
            steps {
                echo "🧰 Running Trivy SCA scan on source code…"
                sh '''
                docker run --rm -v $(pwd):/project aquasec/trivy fs \
                  --scanners vuln \
                  --format json \
                  --output trivy-fs-report.json \
                  /project || true
                '''
                archiveArtifacts artifacts: 'trivy-fs-report.json', fingerprint: true
            }
        }

        stage('Secrets Scan - Gitleaks') {
            steps {
                echo "🔑 Running Gitleaks for secrets detection…"
                sh '''
                docker run --rm -v $(pwd):/repo zricethezav/gitleaks:latest detect \
                  --source=/repo \
                  --report-format json \
                  --report-path /repo/gitleaks-report.json || true
                '''
                archiveArtifacts artifacts: 'gitleaks-report.json', fingerprint: true
            }
        }

        stage('Run Juice Shop') {
            steps {
                echo "🚀 Starting Juice Shop container on port 3000…"
                sh '''
                docker rm -f juice-shop || true
                docker run -d -p 3000:3000 --name juice-shop bkimminich/juice-shop
                sleep 15
                '''
            }
        }

        stage('DAST - OWASP ZAP Baseline') {
            steps {
                echo "🕷️ Running OWASP ZAP baseline scan against Juice Shop…"
                sh '''
                docker run --rm -v $(pwd):/zap/wrk/ owasp/zap2docker-stable zap-baseline.py \
                  -t http://host.docker.internal:3000 \
                  -r zap-report.html || true
                '''
                publishHTML(target: [
                  reportDir: '.',
                  reportFiles: 'zap-report.html',
                  reportName: 'OWASP ZAP Report'
                ])
            }
        }

        stage('Upload ZAP Report to DefectDojo') {
            steps {
                echo "📡 Uploading ZAP report to DefectDojo…"
                sh '''
                if [ -f zap-report.html ]; then
                  curl -X POST "$DEFECTDOJO_URL/api/v2/import-scan/" \
                    -H "Authorization: Token $DEFECTDOJO_API_KEY" \
                    -F "scan_type=ZAP Scan" \
                    -F "file=@zap-report.html" \
                    -F "product_name=$DEFECTDOJO_PRODUCT_NAME" \
                    -F "engagement_name=CI-CD Demo" \
                    -F "scan_date=$(date +%Y-%m-%d)" \
                    -F "active=true" \
                    -F "verified=true" || true
                else
                  echo "zap-report.html not found, skipping DefectDojo upload."
                fi
                '''
            }
        }
    }

    post {
        always {
            echo "🧹 Cleaning up Docker container…"
            sh '''
            docker rm -f juice-shop || true
            '''
        }
    }
}

