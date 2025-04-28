pipeline {
    agent any

    triggers {
        githubPush()  // Automatically triggers this pipeline when code is pushed to GitHub
    }

    environment {
        DOCKER_IMAGE = 'sameer2699/flask-hello-world:latest'
    }

    stages {
        stage('Checkout Code') {
            steps {
                git branch: 'main', url: 'https://github.com/sam99-git/flask-hello-world.git'
            }
        }

        stage('Pre-Commit Scan') {
            steps {
                sh '''
                pip install pre-commit detect-secrets
                pre-commit run --all-files
                '''
            }
        }

        stage('SAST Scan') {
            steps {
                sh '''
                pip install semgrep
                semgrep scan --config auto .
                '''
            }
        }

        stage('Secrets Detection') {
            steps {
                sh '''
                gitleaks detect --source=. --verbose --redact --exit-code 1
                '''
            }
        }

        stage('SCA Dependency Scan') {
            steps {
                sh '''
                trivy fs --scanners vuln,config --exit-code 1 --severity HIGH,CRITICAL .
                '''
            }
        }

        stage('IaC Security Scan') {
            steps {
                sh '''
                pip install checkov
                checkov -d kubernetes/
                '''
            }
        }

        stage('Docker Build & Image Scan') {
            steps {
                sh '''
                docker build -t $DOCKER_IMAGE .
                trivy image $DOCKER_IMAGE --exit-code 1 --severity HIGH,CRITICAL
                '''
            }
        }

        stage('Deploy to Staging (K8s)') {
            steps {
                sh '''
                kubectl apply -f kubernetes/
                '''
            }
        }

        stage('DAST Scan') {
            steps {
                sh '''
                docker run --network host owasp/zap2docker-stable zap-baseline.py -t http://localhost:30007 -r zap-report.html || true
                '''
                archiveArtifacts artifacts: 'zap-report.html'
            }
        }
    }

    post {
        always {
            junit 'zap-report.html'
            archiveArtifacts artifacts: 'zap-report.html', allowEmptyArchive: true
        }
        success {
            echo 'Pipeline completed successfully!'
        }
        failure {
            echo 'Pipeline failed!'
        }
    }
}