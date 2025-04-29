pipeline {
    agent any

    environment {
        SCAN_DIR = "${WORKSPACE}/scan-reports"
        IMAGE_NAME = "sameer2699/flask-hello-world:${env.BUILD_NUMBER}"
        KUBECONFIG_CREDENTIALS = 'kubeconfig-credentials-id' // Replace with your actual credentials ID
        PATH = "${WORKSPACE}/tools:${env.PATH}" // Add tools to the PATH for easy access
    }

    stages {
        stage('Checkout Code') {
            steps {
                checkout([
                    $class: 'GitSCM',
                    branches: [[name: 'main']],
                    extensions: [
                        [$class: 'CleanBeforeCheckout']
                    ],
                    userRemoteConfigs: [
                        [url: 'https://github.com/sam99-git/flask-hello-world.git',
                         credentialsId: 'git-credentials']
                    ]
                ])
            }
        }

        stage('Setup Environment') {
            steps {
                sh '''
                echo "Creating workspace directories..."
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
                python3 -m pip install semgrep

                # Install Gitleaks
                curl -sSfL https://github.com/gitleaks/gitleaks/releases/download/v8.18.1/gitleaks_8.18.1_linux_x64.tar.gz | tar xz -C ${WORKSPACE}/tools

                # Install Trivy
                curl -sfL https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/install.sh | sh -s -- -b ${WORKSPACE}/tools

                # Install Checkov
                python3 -m pip install checkov
                '''
            }
        }

        stage('SAST Scan') {
            steps {
                catchError(buildResult: 'UNSTABLE', stageResult: 'FAILURE') {
                    script {
                        sh "mkdir -p ${SCAN_DIR}"
                        sh "semgrep scan --config auto --sarif --output ${SCAN_DIR}/semgrep-results.sarif ."
                    }
                }
            }
            post {
                always {
                    archiveArtifacts artifacts: "${SCAN_DIR}/semgrep-results.sarif", allowEmptyArchive: true
                }
            }
        }

        stage('Secrets Detection') {
            steps {
                catchError(buildResult: 'UNSTABLE', stageResult: 'FAILURE') {
                    script {
                        sh "mkdir -p ${SCAN_DIR}"
                        sh "gitleaks detect --source=. --report-path=${SCAN_DIR}/gitleaks-report.json --redact"
                    }
                }
            }
            post {
                always {
                    archiveArtifacts artifacts: "${SCAN_DIR}/gitleaks-report.json", allowEmptyArchive: true
                }
            }
        }

        stage('SCA Dependency Scan') {
            steps {
                catchError(buildResult: 'UNSTABLE', stageResult: 'FAILURE') {
                    script {
                        sh "mkdir -p ${SCAN_DIR}"
                        sh "trivy fs --security-checks vuln,config --format sarif --output ${SCAN_DIR}/trivy-deps-results.sarif --exit-code 0 --severity HIGH,CRITICAL ."
                    }
                }
            }
            post {
                always {
                    archiveArtifacts artifacts: "${SCAN_DIR}/trivy-deps-results.sarif", allowEmptyArchive: true
                }
            }
        }

        stage('IaC Security Scan') {
            steps {
                catchError(buildResult: 'UNSTABLE', stageResult: 'FAILURE') {
                    script {
                        sh "mkdir -p ${SCAN_DIR}"
                        sh "checkov -d kubernetes/ --output sarif --output-file-path ${SCAN_DIR}/checkov-results.sarif"
                    }
                }
            }
            post {
                always {
                    archiveArtifacts artifacts: "${SCAN_DIR}/checkov-results.sarif", allowEmptyArchive: true
                }
            }
        }

        stage('Docker Build & Scan') {
            steps {
                catchError(buildResult: 'UNSTABLE', stageResult: 'FAILURE') {
                    script {
                        docker.build("${IMAGE_NAME}")
                        sh "trivy image --format sarif --output ${SCAN_DIR}/trivy-image-results.sarif --exit-code 0 --severity HIGH,CRITICAL ${IMAGE_NAME}"
                    }
                }
            }
            post {
                always {
                    archiveArtifacts artifacts: "${SCAN_DIR}/trivy-image-results.sarif", allowEmptyArchive: true
                }
            }
        }

        stage('Deploy to Staging') {
            steps {
                catchError(buildResult: 'UNSTABLE', stageResult: 'FAILURE') {
                    withCredentials([file(credentialsId: "${KUBECONFIG_CREDENTIALS}", variable: 'KUBECONFIG')]) {
                        sh 'kubectl config set-context --current --namespace=default'
                        sh 'kubectl apply -f kubernetes/'
                    }
                }
            }
        }

        stage('DAST Scan') {
            steps {
                catchError(buildResult: 'UNSTABLE', stageResult: 'FAILURE') {
                    script {
                        sh "mkdir -p ${SCAN_DIR}"
                        sh '''
                        docker run --rm \
                            -v ${SCAN_DIR}:/zap/reports \
                            -t owasp/zap2docker-stable \
                            zap-baseline.py \
                            -t http://localhost:30007 \
                            -r zap-report.html \
                            -x zap-report.xml
                        '''
                    }
                }
            }
            post {
                always {
                    archiveArtifacts artifacts: "${SCAN_DIR}/zap-report.*", allowEmptyArchive: true
                }
            }
        }

        stage('Smoke Tests') {
            steps {
                catchError(buildResult: 'UNSTABLE', stageResult: 'FAILURE') {
                    retry(3) {
                        sleep 5
                        sh 'curl -sSf http://localhost:30007/health | grep -q \'"status":"OK"\''
                    }
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