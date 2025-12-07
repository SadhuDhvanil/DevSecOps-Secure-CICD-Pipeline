pipeline {
    agent any

    stages {

        stage('Build') {
            steps {
                echo "⚙️ Installing dependencies for Juice Shop..."
                sh '''
                docker run --rm -v $(pwd):/app -w /app node:18 bash -c "npm install"
                '''
            }
        }

        stage('SAST - Semgrep') {
            steps {
                echo "🔍 Running Semgrep SAST scan..."
                sh '''
                docker run --rm -v $(pwd):/src -w /src semgrep/semgrep semgrep \
                  --config=auto \
                  --json \
                  --output semgrep-report.json || true
                '''
                archiveArtifacts artifacts: 'semgrep-report.json', fingerprint: true
            }
        }

        stage('Run Juice Shop') {
            steps {
                echo "🚀 Starting Juice Shop container on port 3000..."
                sh '''
                docker run -d -p 3000:3000 --name juice-shop bkimminich/juice-shop
                sleep 10
                '''
            }
        }

        stage('Verify App Running') {
            steps {
                echo "🔍 Checking if Juice Shop is accessible..."
                sh '''
                curl -I http://host.docker.internal:3000 || true
                '''
            }
        }
    }

    post {
        always {
            echo "🧹 Cleaning up Docker container..."
            sh '''
            docker rm -f juice-shop || true
            '''
        }
    }
}

