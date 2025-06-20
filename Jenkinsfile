pipeline {
    agent any

    environment {
        SCAN_DIR = "${WORKSPACE}/scan-reports"
        IMAGE_NAME = "sameer2699/flask-hello-world:${env.BUILD_NUMBER}"
        KUBECONFIG_CREDENTIALS = 'kubeconfig-credentials-id'
        PATH = "${WORKSPACE}/tools:$HOME/.local/bin:${env.PATH}"
        DOCKER_PASSWORD = 'docker-credentials'
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
                tree -L 3 ${WORKSPACE} || true
                '''
            }
        }

        stage('Install Security Tools') {
            steps {
                sh '''
                export PATH=$HOME/.local/bin:$PATH

                python3 -m pip install --upgrade pip --user
                python3 -m pip install --user semgrep checkov

                # Symlink to tools directory for consistent usage
                ln -sf $HOME/.local/bin/semgrep ${WORKSPACE}/tools/semgrep
                ln -sf $HOME/.local/bin/checkov ${WORKSPACE}/tools/checkov

                # Gitleaks binary
                curl -sSfL https://github.com/gitleaks/gitleaks/releases/download/v8.18.1/gitleaks_8.18.1_linux_x64.tar.gz | tar xz -C ${WORKSPACE}/tools

                # Trivy binary
                curl -sfL https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/install.sh | sh -s -- -b ${WORKSPACE}/tools

                # Verify installations
                ${WORKSPACE}/tools/semgrep --version || echo "Semgrep installation failed"
                ${WORKSPACE}/tools/checkov --version || echo "Checkov installation failed"
                ${WORKSPACE}/tools/trivy --version || echo "Trivy installation failed"
                ${WORKSPACE}/tools/gitleaks version || echo "Gitleaks installation failed"
                '''
            }
        }

        stage('SAST Scan') {
            steps {
                sh "${WORKSPACE}/tools/semgrep scan --config auto --sarif --output ${SCAN_DIR}/semgrep-results.sarif ."
            }
            post {
                always {
                    archiveArtifacts artifacts: "scan-reports/semgrep-results.sarif", allowEmptyArchive: true
                }
            }
        }

        stage('Secrets Detection') {
            steps {
                sh "${WORKSPACE}/tools/gitleaks detect --source=. --report-path=${SCAN_DIR}/gitleaks-report.json --redact"
            }
            post {
                always {
                    archiveArtifacts artifacts: "scan-reports/gitleaks-report.json", allowEmptyArchive: true
                }
            }
        }

        stage('SCA Dependency Scan') {
            steps {
                sh "${WORKSPACE}/tools/trivy fs --scanners vuln,misconfig --format sarif --output ${SCAN_DIR}/trivy-deps-results.sarif --exit-code 0 --severity HIGH,CRITICAL ."
            }
            post {
                always {
                    archiveArtifacts artifacts: "scan-reports/trivy-deps-results.sarif", allowEmptyArchive: true
                }
            }
        }

        stage('IaC Security Scan') {
            steps {
                sh "${WORKSPACE}/tools/checkov -d kubernetes/ --output sarif --output-file-path ${SCAN_DIR}/checkov-results.sarif"
            }
            post {
                always {
                    archiveArtifacts artifacts: "scan-reports/checkov-results.sarif", allowEmptyArchive: true
                }
            }
        }

        stage('Docker Build & Scan') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'docker-credentials',
                    usernameVariable: 'DOCKER_USERNAME',
                    passwordVariable: 'DOCKER_PASSWORD'
                )]) {
                    sh '''
                        # Build, tag, and push the image
                        docker build -t ${IMAGE_NAME} .
                        echo "$DOCKER_PASSWORD" | docker login -u "$DOCKER_USERNAME" --password-stdin
                        docker push ${IMAGE_NAME}

                        # 👇 Get the SHA256 digest of the pushed image
                        DIGEST=$(docker inspect --format='{{index .RepoDigests 0}}' ${IMAGE_NAME} | cut -d'@' -f2)
                        echo "Image digest: $DIGEST"

                        # 👇 Replace placeholder in deployment.yaml
                        sed -i "s|IMAGE_DIGEST|${DIGEST}|g" kubernetes/deployment.yaml
                        
                        # Optional: Update imagePullPolicy to "Always" for compliance
                        sed -i 's|imagePullPolicy: IfNotPresent|imagePullPolicy: Always|g' kubernetes/deployment.yaml
                        echo "Updated deployment.yaml:"
                        cat kubernetes/deployment.yaml
                    '''
                }
            }

            post {
                always {
                    archiveArtifacts artifacts: "scan-reports/trivy-image-results.sarif", allowEmptyArchive: true
                }
            }
        }
        stage('Deploy to Staging') {
            steps {
                withCredentials([file(credentialsId: "${KUBECONFIG_CREDENTIALS}", variable: 'KUBECONFIG_FILE')]) {
                    sh '''
                    export KUBECONFIG=$KUBECONFIG_FILE

                    # Just to verify context
                    kubectl config current-context
                    kubectl get nodes
                    kubectl apply --validate=false -f kubernetes/serviceaccount.yaml
                    kubectl apply --validate=false -f kubernetes/namespace.yaml
                    kubectl apply --validate=false -f kubernetes/deployment.yaml
                    kubectl apply --validate=false -f kubernetes/service.yaml
                    '''
                }
            }
        }

    }

    post {
        always {
            
            // Archive all scan reports
            archiveArtifacts artifacts: 'scan-reports/**/*', allowEmptyArchive: true
        }
    }
}