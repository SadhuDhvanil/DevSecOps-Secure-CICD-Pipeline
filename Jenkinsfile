pipeline {
    agent any

    options {
        skipDefaultCheckout()
    }

    environment {
        WORKDIR = "${WORKSPACE}/app"
        DEFECTDOJO_URL = "http://host.docker.internal:8082"
        DEFECTDOJO_API_KEY = credentials('defectdojo_api_token')
        DEFECTDOJO_PRODUCT_NAME = "OWASP Juice Shop Pipeline"
    }

    stages {

        stage('Checkout Code') {
            steps {
                echo "📦 Cloning repository..."
                sh '''
                rm -rf app || true
                git clone https://github.com/SadhuDhvanil/DevSecOps-Secure-CICD-Pipeline.git app
                ls -la app
                '''
            }
        }

stage('Build (npm install)') {
    steps {
        echo "⚙️ Installing backend dependencies (Node 22)..."
        sh """
        docker run --rm \
          -v "$WORKSPACE":/workspace \
          -w /workspace \
          node:22 bash -c "npm install --legacy-peer-deps"
        """
    }
}

        stage('SAST - Semgrep') {
            steps {
                echo "🔍 Running Semgrep scan..."
                dir('app') {
                    sh '''
                    docker run --rm \
                      -v "$PWD":/src -w /src \
                      semgrep/semgrep semgrep \
                        --config=auto \
                        --json \
                        --output semgrep-report.json || true

                    # guarantee file exists
                    [ ! -f semgrep-report.json ] && echo '{}' > semgrep-report.json
                    '''
                }
                archiveArtifacts artifacts: 'app/semgrep-report.json', fingerprint: true
            }
        }

        stage('SCA - Trivy FS Scan') {
            steps {
                echo "🧰 Running Trivy filesystem scan..."
                sh '''
                docker run --rm \
                  -v "$WORKDIR":/src \
                  aquasec/trivy fs /src \
                  --format json \
                  --output /src/trivy-fs-report.json || true
                '''
                archiveArtifacts artifacts: 'app/trivy-fs-report.json', fingerprint: true
            }
        }

        stage('Secrets Scan - Gitleaks') {
            steps {
                echo "🔑 Running Gitleaks..."
                sh '''
                docker run --rm \
                  -v "$WORKDIR":/repo \
                  zricethezav/gitleaks detect \
                    --source=/repo \
                    --report-format json \
                    --report-path /repo/gitleaks-report.json || true
                '''
                archiveArtifacts artifacts: 'app/gitleaks-report.json', fingerprint: true
            }
        }

        stage('Run Juice Shop') {
            steps {
                echo "🚀 Starting Juice Shop container..."
                sh '''
                docker rm -f juice-shop || true
                docker run -d -p 3000:3000 --name juice-shop bkimminich/juice-shop
                sleep 15
                '''
            }
        }

        stage('DAST - ZAP Baseline Scan') {
            steps {
                echo "🕷️ Running OWASP ZAP baseline..."
                dir('app') {
                    sh '''
                    docker run --rm \
                      -v "$PWD":/zap/wrk \
                      owasp/zap2docker-stable zap-baseline.py \
                        -t http://host.docker.internal:3000 \
                        -r zap-report.html || true
                    '''
                }

                publishHTML(target: [
                    reportDir: "app",
                    reportFiles: "zap-report.html",
                    reportName: "OWASP ZAP Report"
                ])
            }
        }

        stage('Upload ZAP Report to DefectDojo') {
            steps {
                echo "📡 Uploading report to DefectDojo..."
                dir('app') {
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
                      echo "❗ zap-report.html missing — skipping upload"
                    fi
                    '''
                }
            }
        }
    }

    post {
        always {
            echo "🧹 Cleaning up..."
            sh "docker rm -f juice-shop || true"
        }
    }
}

