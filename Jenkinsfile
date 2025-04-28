pipeline {
    agent any

    triggers {
        pollSCM('* * * * *')  // More reliable than githubPush for most setups
    }

    environment {
        DOCKER_IMAGE = 'sameer2699/flask-hello-world:${env.BUILD_NUMBER}'
        KUBE_NAMESPACE = 'flask-staging'
        SCAN_DIR = "${WORKSPACE}/scan-reports"
    }

    stages {
        stage('Setup Environment') {
            steps {
                sh '''
                mkdir -p ${SCAN_DIR}
                python -m pip install --upgrade pip
                '''
            }
        }

        stage('Checkout Code') {
            steps {
                checkout([
                    $class: 'GitSCM',
                    branches: [[name: 'main']],
                    extensions: [[
                        $class: 'CleanBeforeCheckout',
                        deleteUntrackedNestedRepositories: true
                    ]],
                    userRemoteConfigs: [[
                        url: 'https://github.com/sam99-git/flask-hello-world.git'
                    ]]
                ])
            }
        }

        stage('Install Security Tools') {
            steps {
                sh '''
                # Install Semgrep
                python -m pip install semgrep

                # Install Gitleaks
                curl -sSfL https://github.com/gitleaks/gitleaks/releases/download/v8.18.1/gitleaks_8.18.1_linux_x64.tar.gz | tar xz -C /usr/local/bin/

                # Install Trivy
                curl -sfL https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/install.sh | sh -s -- -b /usr/local/bin

                # Install Checkov
                python -m pip install checkov
                '''
            }
        }

        stage('SAST Scan') {
            steps {
                sh '''
                semgrep scan --config auto --error-on-findings --sarif --output ${SCAN_DIR}/semgrep-results.sarif .
                '''
            }
            post {
                always {
                    archiveArtifacts artifacts: '${SCAN_DIR}/semgrep-results.sarif'
                }
            }
        }

        stage('Secrets Detection') {
            steps {
                sh '''
                gitleaks detect --source=. --report-path=${SCAN_DIR}/gitleaks-report.json --redact
                '''
            }
            post {
                always {
                    archiveArtifacts artifacts: '${SCAN_DIR}/gitleaks-report.json'
                }
            }
        }

        stage('SCA Dependency Scan') {
            steps {
                sh '''
                trivy fs --security-checks vuln,config --format sarif --output ${SCAN_DIR}/trivy-deps-results.sarif --exit-code 0 --severity HIGH,CRITICAL .
                '''
            }
            post {
                always {
                    archiveArtifacts artifacts: '${SCAN_DIR}/trivy-deps-results.sarif'
                }
            }
        }

        stage('IaC Security Scan') {
            steps {
                sh '''
                checkov -d kubernetes/ --output sarif --output-file-path ${SCAN_DIR}/checkov-results.sarif
                '''
            }
            post {
                always {
                    archiveArtifacts artifacts: '${SCAN_DIR}/checkov-results.sarif'
                }
            }
        }

        stage('Docker Build & Scan') {
            steps {
                script {
                    docker.build("${DOCKER_IMAGE}")
                    sh '''
                    trivy image --format sarif --output ${SCAN_DIR}/trivy-image-results.sarif --exit-code 0 --severity HIGH,CRITICAL ${DOCKER_IMAGE}
                    '''
                }
            }
            post {
                always {
                    archiveArtifacts artifacts: '${SCAN_DIR}/trivy-image-results.sarif'
                }
            }
        }

        stage('Deploy to Staging') {
            steps {
                sh '''
                kubectl config set-context --current --namespace=${KUBE_NAMESPACE}
                kubectl apply -f kubernetes/ --dry-run=server
                kubectl apply -f kubernetes/
                '''
            }
        }

        stage('DAST Scan') {
            steps {
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
            post {
                always {
                    archiveArtifacts artifacts: '${SCAN_DIR}/zap-report.*'
                }
            }
        }

        stage('Smoke Tests') {
            steps {
                sh '''
                curl -sSf http://localhost:30007/health | grep -q '"status":"OK"'
                '''
            }
        }
    }

    post {
        always {
            junit allowEmptyResults: true, testResults: '${SCAN_DIR}/*.xml'
            archiveArtifacts artifacts: '${SCAN_DIR}/*.*', allowEmptyArchive: true
            cleanWs()
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