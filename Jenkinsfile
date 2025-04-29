pipeline {
    agent any

    environment {
        SCAN_DIR = "${WORKSPACE}/scan-reports"
        IMAGE_NAME = "sameer2699/flask-hello-world:${env.BUILD_NUMBER}"
        KUBECONFIG_CREDENTIALS = 'kubeconfig-credentials-id'
        PATH = "${WORKSPACE}/tools:${env.PATH}"
    }

    stages {
        stage('Checkout Code') {
            steps {
                checkout scm
            }
        }

        stage('Setup Environment') {
            steps {
                sh '''
                mkdir -p ${SCAN_DIR} ${WORKSPACE}/tools
                echo "Directory structure:"
                tree -L 3 ${WORKSPACE}
                '''
            }
        }

        stage('Install Security Tools') {
            steps {
                sh '''
                # Install Semgrep
                python3 -m pip install --target=${WORKSPACE}/tools semgrep
                echo "alias semgrep='python3 ${WORKSPACE}/tools/semgrep'" >> ~/.bashrc

                # Install Checkov
                python3 -m pip install --target=${WORKSPACE}/tools checkov
                echo "alias checkov='python3 ${WORKSPACE}/tools/checkov'" >> ~/.bashrc

                # Install Gitleaks
                curl -sSfL https://github.com/gitleaks/gitleaks/releases/download/v8.18.1/gitleaks_8.18.1_linux_x64.tar.gz | tar xz -C ${WORKSPACE}/tools

                # Install Trivy
                curl -sfL https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/install.sh | sh -s -- -b ${WORKSPACE}/tools

                # Verify installations
                source ~/.bashrc
                semgrep --version || echo "Semgrep installation failed"
                ${WORKSPACE}/tools/trivy --version || echo "Trivy installation failed"
                checkov --version || echo "Checkov installation failed"
                '''
            }
        }

        stage('SAST Scan') {
            steps {
                sh "semgrep scan --config auto --sarif --output ${SCAN_DIR}/semgrep-results.sarif ."
            }
            post {
                always {
                    archiveArtifacts artifacts: "scan-reports/semgrep-results.sarif", allowEmptyArchive: true
                }
            }
        }

        stage('Secrets Detection') {
            steps {
                sh "gitleaks detect --source=. --report-path=${SCAN_DIR}/gitleaks-report.json --redact"
            }
            post {
                always {
                    archiveArtifacts artifacts: "scan-reports/gitleaks-report.json", allowEmptyArchive: true
                }
            }
        }

        stage('SCA Dependency Scan') {
            steps {
                sh "trivy fs --scanners vuln,misconfig --format sarif --output ${SCAN_DIR}/trivy-deps-results.sarif --exit-code 0 --severity HIGH,CRITICAL ."
            }
            post {
                always {
                    archiveArtifacts artifacts: "scan-reports/trivy-deps-results.sarif", allowEmptyArchive: true
                }
            }
        }

        stage('IaC Security Scan') {
            steps {
                sh "checkov -d kubernetes/ --output sarif --output-file-path ${SCAN_DIR}/checkov-results.sarif"
            }
            post {
                always {
                    archiveArtifacts artifacts: "scan-reports/checkov-results.sarif", allowEmptyArchive: true
                }
            }
        }

        stage('Docker Build & Scan') {
            steps {
                sh '''
                docker build -t ${IMAGE_NAME} .
                trivy image --format sarif --output ${SCAN_DIR}/trivy-image-results.sarif --exit-code 0 --severity HIGH,CRITICAL ${IMAGE_NAME}
                '''
            }
            post {
                always {
                    archiveArtifacts artifacts: "scan-reports/trivy-image-results.sarif", allowEmptyArchive: true
                }
            }
        }

        stage('Deploy to Staging') {
            steps {
                withCredentials([file(credentialsId: "${KUBECONFIG_CREDENTIALS}", variable: 'KUBECONFIG')]) {
                    sh '''
                    kubectl apply -f kubernetes/namespace.yaml
                    kubectl apply -f kubernetes/ --validate=false
                    '''
                }
            }
        }
    }

    post {
        always {
            junit allowEmptyResults: true, testResults: "${SCAN_DIR}/*.xml"
            archiveArtifacts artifacts: "${SCAN_DIR}/*.*", allowEmptyArchive: true
        }
        success {
            slackSend(color: 'good', message: "Pipeline SUCCESS: ${env.JOB_NAME} #${env.BUILD_NUMBER}")
        }
        failure {
            slackSend(color: 'danger', message: "Pipeline FAILED: ${env.JOB_NAME} #${env.BUILD_NUMBER}")
        }
        unstable {
            slackSend(color: 'warning', message: "Pipeline UNSTABLE: ${env.JOB_NAME} #${env.BUILD_NUMBER}")
        }
    }
}