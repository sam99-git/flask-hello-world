pipeline {
    agent any

    environment {
        SCAN_DIR = "${WORKSPACE}/scan-reports"
        IMAGE_NAME = "sameer2699/flask-hello-world:4"
        KUBECONFIG_CREDENTIALS = 'kubeconfig-credentials-id' // replace with your actual credentials ID
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
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
                        sh "trufflehog filesystem --directory . --json > ${SCAN_DIR}/secrets.json || true"
                    }
                }
            }
            post {
                always {
                    archiveArtifacts artifacts: "${SCAN_DIR}/secrets.json", allowEmptyArchive: true
                }
            }
        }

        stage('SCA Dependency Scan') {
            steps {
                catchError(buildResult: 'UNSTABLE', stageResult: 'FAILURE') {
                    script {
                        sh "mkdir -p ${SCAN_DIR}"
                        sh "safety check -r requirements.txt --json > ${SCAN_DIR}/safety-results.json || true"
                    }
                }
            }
            post {
                always {
                    archiveArtifacts artifacts: "${SCAN_DIR}/safety-results.json", allowEmptyArchive: true
                }
            }
        }

        stage('IaC Security Scan') {
            steps {
                catchError(buildResult: 'UNSTABLE', stageResult: 'FAILURE') {
                    script {
                        sh "mkdir -p ${SCAN_DIR}"
                        sh "checkov -d . --output sarif --output-file-path ${SCAN_DIR}/checkov-results.sarif || true"
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
                        sh "docker build -t ${IMAGE_NAME} ."
                        sh "docker scan ${IMAGE_NAME} || true"
                    }
                }
            }
        }

        stage('Deploy to Staging') {
            steps {
                catchError(buildResult: 'UNSTABLE', stageResult: 'FAILURE') {
                    withCredentials([file(credentialsId: "${KUBECONFIG_CREDENTIALS}", variable: 'KUBECONFIG')]) {
                        sh 'kubectl config set-context --current --namespace=default'
                        sh 'kubectl apply -f k8s/'
                    }
                }
            }
        }

        stage('DAST Scan') {
            steps {
                catchError(buildResult: 'UNSTABLE', stageResult: 'FAILURE') {
                    script {
                        sh "mkdir -p ${SCAN_DIR}"
                        sh "docker run --rm -v ${SCAN_DIR}:/zap/reports -t owasp/zap2docker-stable zap-baseline.py -t http://localhost:30007 -r zap-report.html -x zap-report.xml || true"
                    }
                }
            }
            post {
                always {
                    archiveArtifacts artifacts: "${SCAN_DIR}/*.*", allowEmptyArchive: true
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
            junit '**/test-results/*.xml'
            archiveArtifacts artifacts: "${SCAN_DIR}/*.*", allowEmptyArchive: true
        }
    }
}
