pipeline {
    agent any

    options {
        skipDefaultCheckout()
    }

    environment {
        WORKDIR = "${WORKSPACE}"               // ✔️ Correct directory
        DEFECTDOJO_URL = "http://host.docker.internal:8082"
        DEFECTDOJO_API_KEY = credentials('defectdojo_api_token')
        DEFECTDOJO_PRODUCT_NAME = "OWASP Juice Shop Pipeline"
    }

    stages {

        stage('Checkout Code') {
            steps {
                echo "📦 Cloning repository..."
                sh '''
                rm -rf repo || true
                git clone https://github.com/SadhuDhvanil/DevSecOps-Secure-CICD-Pipeline.git repo
                ls -la repo
                '''
            }
        }

stage('Build (npm install)') {
    steps {
        echo "⚙️ Installing backend dependencies (Node 22)..."
        sh '''
            docker run --rm \
                -v "$WORKSPACE":/app \
                -w /app \
                node:22 bash -c "npm install --legacy-peer-deps"
        '''
    }
}

        // ---------- SAST ----------
        stage('SAST - Semgrep') {
            steps {
                echo "🔍 Running Semgrep scan..."
                sh '''
                cd repo
                docker run --rm \
                  -v "$PWD":/src -w /src \
                  semgrep/semgrep semgrep \
                    --config=auto \
                    --json \
                    --output semgrep-report.json || true

                # Ensure report exists
                if [ ! -f repo/semgrep-report.json ]; then echo "{}" > repo/semgrep-report.json; fi
                '''
                archiveArtifacts artifacts: "repo/semgrep-report.json", fingerprint: true
            }
        }

        // ---------- SCA ----------
        stage('SCA - Trivy (Filesystem Scan)') {
            steps {
                echo "📦 Running Trivy FS Scan..."
                sh '''
                cd repo
                docker run --rm \
                  -v "$PWD":/workspace \
                  aquasec/trivy fs /workspace \
                  --format json \
                  --output trivy-fs-report.json || true
                '''
                archiveArtifacts artifacts: "repo/trivy-fs-report.json", fingerprint: true
            }
        }

        // ---------- SECRETS ----------
        stage('Secrets Scan - Gitleaks') {
            steps {
                echo "🔑 Running Gitleaks..."
                sh '''
                cd repo
                docker run --rm \
                  -v "$PWD":/workspace \
                  zricethezav/gitleaks detect \
                    --source=/workspace \
                    --report-format json \
                    --report-path gitleaks-report.json || true
                '''
                archiveArtifacts artifacts: "repo/gitleaks-report.json", fingerprint: true
            }
        }

        // ---------- START JUICE SHOP ----------
        stage('Run Juice Shop') {
            steps {
                echo "🚀 Starting Juice Shop..."
                sh '''
                docker rm -f juice-shop || true
                docker run -d -p 3000:3000 --name juice-shop bkimminich/juice-shop
                sleep 20
                '''
            }
        }

        // ---------- DAST ----------
        stage('DAST - OWASP ZAP') {
            steps {
                echo "🕷️ Running ZAP Baseline Scan..."
                sh '''
                cd repo
                docker run --rm \
                  -v "$PWD":/zap \
                  owasp/zap2docker-stable zap-baseline.py \
                    -t http://host.docker.internal:3000 \
                    -r zap-report.html || true
                '''
                publishHTML(target: [
                    reportDir: "repo",
                    reportFiles: "zap-report.html",
                    reportName: "OWASP ZAP Report"
                ])
            }
        }

        // ---------- DEFECTDOJO ----------
        stage('Upload to DefectDojo') {
            steps {
                echo "📡 Uploading reports to DefectDojo..."

                sh '''
                cd repo

                echo "Uploading Semgrep..."
                curl -X POST "$DEFECTDOJO_URL/api/v2/import-scan/" \
                  -H "Authorization: Token $DEFECTDOJO_API_KEY" \
                  -F scan_type="Semgrep JSON Report" \
                  -F product_name="$DEFECTDOJO_PRODUCT_NAME" \
                  -F engagement_name="CI-CD Demo" \
                  -F file=@semgrep-report.json || true

                echo "Uploading Trivy..."
                curl -X POST "$DEFECTDOJO_URL/api/v2/import-scan/" \
                  -H "Authorization: Token $DEFECTDOJO_API_KEY" \
                  -F scan_type="Trivy Scan" \
                  -F product_name="$DEFECTDOJO_PRODUCT_NAME" \
                  -F engagement_name="CI-CD Demo" \
                  -F file=@trivy-fs-report.json || true

                echo "Uploading Gitleaks..."
                curl -X POST "$DEFECTDOJO_URL/api/v2/import-scan/" \
                  -H "Authorization: Token $DEFECTDOJO_API_KEY" \
                  -F scan_type="Gitleaks Scan" \
                  -F product_name="$DEFECTDOJO_PRODUCT_NAME" \
                  -F engagement_name="CI-CD Demo" \
                  -F file=@gitleaks-report.json || true

                echo "Uploading ZAP..."
                curl -X POST "$DEFECTDOJO_URL/api/v2/import-scan/" \
                  -H "Authorization: Token $DEFECTDOJO_API_KEY" \
                  -F scan_type="ZAP Scan" \
                  -F product_name="$DEFECTDOJO_PRODUCT_NAME" \
                  -F engagement_name="CI-CD Demo" \
                  -F file=@zap-report.html || true
                '''
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

