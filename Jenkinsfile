pipeline {
    agent any
   
    // Tools Configuration
    tools {
        jdk "jdk-17.0"
        maven "maven-3.9"
        nodejs "nodejs-23.8"
    }
   
    // Environment Variables
    environment {
        SCANNER_HOME = tool 'sonar-scanner'
        FRONTEND_IMAGE_NAME = 'three-tier-todo-frontend'
        BACKEND_IMAGE_NAME = "three-tier-todo-backend"
        IMAGE_TAG = "${BUILD_NUMBER}"
        DOCKER_REGISTRY = '17rj'  // your Docker Hub username / registry
        K8S_NAMESPACE = 'threetierapp'
        RECIPIENTS = 'your-email@example.com, team@example.com'  // add your emails here
    }
   
    stages {
        stage('Clean Workspace') {
            steps {
                cleanWs()
            }
        }
       
        stage('Checkout Code') {
            steps {
                git branch: 'main', url: 'https://github.com/17J/Three-Tier-Todo-Application.git'
            }
        }
       
        stage('Compilation of Codes') {
            parallel {
                stage('Frontend Compilation') {
                    steps {
                        dir('frontend') {
                            sh 'find . -name "*.js" -exec node --check {} +'
                        }
                    }
                }
                stage('Backend Compilation') {
                    steps {
                        dir('backend') {
                            sh "mvn clean compile"
                        }
                    }
                }
            }
        }
 
        stage('Gitleaks Checking') {
            steps {
                sh "gitleaks detect --no-git -v --redact --exit-code 1 --report-path leaks_report.json"
            }
        }
       
        stage('Snyk SCA Scan') {
            parallel {
                stage('Snyk Frontend Scan') {
                    steps {
                        dir('frontend') {
                            withCredentials([string(credentialsId: 'snyk-token', variable: 'SNYK_TOKEN')]) {
                                sh '''
                                    snyk auth $SNYK_TOKEN
                                    snyk test --json-file-output=snyk-frontend-report.json || true
                                    snyk monitor --project-name=three-tier-todo-frontend || true
                                '''
                            }
                        }
                    }
                }
                stage('Snyk Backend Scan') {
                    steps {
                        dir('backend') {
                            withCredentials([string(credentialsId: 'snyk-token', variable: 'SNYK_TOKEN')]) {
                                sh '''
                                    snyk auth $SNYK_TOKEN
                                    snyk test --json-file-output=snyk-backend-report.json || true
                                    snyk monitor --project-name=three-tier-todo-backend || true
                                '''
                            }
                        }
                    }
                }
            }
        }
       
        stage('Sonarqube Code Analysis') {
            steps {
                withSonarQubeEnv("sonar") {
                    sh '''
                        $SCANNER_HOME/bin/sonar-scanner \
                        -Dsonar.projectName=threetiertodo \
                        -Dsonar.projectKey=threetiertodo \
                        -Dsonar.java.binaries=backend/target/classes \
                        -Dsonar.sources=.,frontend/src,backend/src/main/java \
                        -Dsonar.tests=backend/src/test/java,frontend/src/tests  # adjust if needed
                    '''
                }
            }
        }
       
        stage('Quality Gate') {
            steps {
                timeout(time: 1, unit: 'HOURS') {
                    waitForQualityGate abortPipeline: false, credentialsId: 'sonarqube'
                }
            }
        }
       
        stage('Trivy FS Scan') {
            steps {
                parallel {
                    stage('Frontend FS') { sh "trivy fs --format table -o trivy-fs-frontend.html frontend/" }
                    stage('Backend FS')  { sh "trivy fs --format table -o trivy-fs-backend.html backend/" }
                }
            }
        }
       
        stage('Docker Build & Push') {
            parallel {
                stage('Frontend Docker Build') {
                    steps {
                        dir('frontend') {
                            withDockerRegistry(credentialsId: 'docker-creds', toolName: 'docker') {
                                sh '''
                                    docker build --pull -t ${DOCKER_REGISTRY}/${FRONTEND_IMAGE_NAME}:${IMAGE_TAG} .
                                    docker push ${DOCKER_REGISTRY}/${FRONTEND_IMAGE_NAME}:${IMAGE_TAG}
                                '''
                            }
                        }
                    }
                }
                stage('Backend Docker Build') {
                    steps {
                        dir('backend') {
                            withDockerRegistry(credentialsId: 'docker-creds', toolName: 'docker') {
                                sh '''
                                    docker build --pull -t ${DOCKER_REGISTRY}/${BACKEND_IMAGE_NAME}:${IMAGE_TAG} .
                                    docker push ${DOCKER_REGISTRY}/${BACKEND_IMAGE_NAME}:${IMAGE_TAG}
                                '''
                            }
                        }
                    }
                }
            }
        }
       
        stage('Docker Image Scan') {
            parallel {
                stage('Trivy Image Scan') {
                    steps {
                        sh "trivy image --format table -o trivy-frontend-image.html ${DOCKER_REGISTRY}/${FRONTEND_IMAGE_NAME}:${IMAGE_TAG}"
                        sh "trivy image --format table -o trivy-backend-image.html ${DOCKER_REGISTRY}/${BACKEND_IMAGE_NAME}:${IMAGE_TAG}"
                    }
                }
                stage('Snyk Container Scan') {
                    steps {
                        withCredentials([string(credentialsId: 'snyk-token', variable: 'SNYK_TOKEN')]) {
                            sh '''
                                snyk auth $SNYK_TOKEN
                                snyk container test ${DOCKER_REGISTRY}/${FRONTEND_IMAGE_NAME}:${IMAGE_TAG} --json-file-output=snyk-frontend-container.json || true
                                snyk container test ${DOCKER_REGISTRY}/${BACKEND_IMAGE_NAME}:${IMAGE_TAG} --json-file-output=snyk-backend-container.json || true
                            '''
                        }
                    }
                }
            }
        }
       
        stage('Kubernetes Deploy') {
            steps {
                dir('K8s') {
                    withKubeConfig([credentialsId: 'kube-cred']) {  // assuming full kubeconfig in credential
                        sh '''
                            set -e
                            kubectl apply -f backend-ds-service.yml -n ${K8S_NAMESPACE}
                            kubectl apply -f db-ds-service.yml -n ${K8S_NAMESPACE}
                            kubectl apply -f frontend-ds-service.yml -n ${K8S_NAMESPACE}
                            kubectl apply -f ingress.yml -n ${K8S_NAMESPACE}
                            kubectl apply -f secrets-configmap.yml -n ${K8S_NAMESPACE}
                            
                            # Wait for deployments to be ready
                            kubectl wait --for=condition=available --timeout=120s deployment/backend-deployment -n ${K8S_NAMESPACE}
                            kubectl wait --for=condition=available --timeout=120s deployment/db-deployment -n ${K8S_NAMESPACE} || true  # if db is StatefulSet, adjust
                            kubectl wait --for=condition=available --timeout=120s deployment/frontend-deployment -n ${K8S_NAMESPACE}
                        '''
                    }
                }
            }
        }
       
        stage('Retrieve Application URL') {
            steps {
                script {
                    withKubeConfig([credentialsId: 'kube-cred']) {
                        // Assuming ingress has hostname in status (common for AWS ALB)
                        env.K8S_URL = sh(script: "kubectl get ingress frontend-ingress -n ${K8S_NAMESPACE} -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'", returnStdout: true).trim()
                        if (!env.K8S_URL) {
                            error "Ingress hostname not found! Check ingress status."
                        }
                    }
                }
                echo "Frontend URL: https://${env.K8S_URL}"  // usually HTTPS with Ingress
            }
        }
       
        stage('Kubernetes Verify') {
            steps {
                withKubeConfig([credentialsId: 'kube-cred']) {
                    sh '''
                        kubectl get pods -n ${K8S_NAMESPACE}
                        kubectl get svc -n ${K8S_NAMESPACE}
                        kubectl get ingress -n ${K8S_NAMESPACE}
                    '''
                }
            }
        }
       
        stage('OWASP ZAP Baseline Scan') {
            steps {
                script {
                    // Using official ZAP Docker for baseline (passive + spider, CI friendly)
                    sh '''
                        docker run --rm \
                          -v $(pwd):/zap/wrk/:rw \
                          -t ghcr.io/zaproxy/zaproxy:stable \
                          zap-baseline.py \
                          -t https://${K8S_URL} \
                          -r zap-baseline-report.html \
                          -x zap-baseline-report.xml \
                          -I  # ignore failures for now, or remove to fail build
                    '''
                    archiveArtifacts artifacts: 'zap-baseline-report.*', allowEmptyArchive: true
                }
            }
        }
    }
   
    post {
        always {
            echo 'Collecting Falco Security Logs...'
            sh 'kubectl logs -l app=falco -n falco --tail=100 || true'  // adjust ns if different
           
            archiveArtifacts artifacts: '**/snyk-*-report.json, **/trivy-*.html, **/zap-*.html', allowEmptyArchive: true
        }
        success {
            emailext(
                subject: "✅ SUCCESS: Jenkins Build #${BUILD_NUMBER} - ${JOB_NAME}",
                body: """
                <h2>✅ Build Successful!</h2>
                <p>Jenkins Job: ${JOB_NAME}</p>
                <p>Build Number: ${BUILD_NUMBER}</p>
                <p>App URL: https://${K8S_URL}</p>
                <p>Check logs: <a href="${BUILD_URL}console">here</a></p>
                """,
                to: "${RECIPIENTS}",
                mimeType: "text/html"
            )
        }
        failure {
            emailext(
                subject: "❌ FAILURE: Jenkins Build #${BUILD_NUMBER} - ${JOB_NAME}",
                body: """
                <h2>❌ Build Failed!</h2>
                <p>Jenkins Job: ${JOB_NAME}</p>
                <p>Build Number: ${BUILD_NUMBER}</p>
                <p>Check logs: <a href="${BUILD_URL}console">here</a></p>
                """,
                to: "${RECIPIENTS}",
                mimeType: "text/html"
            )
        }
    }
}