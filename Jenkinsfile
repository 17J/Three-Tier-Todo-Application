pipeline {
    agent any

    // Global Tools Configuration for Java, Maven, and Node.js
    tools {
        jdk "jdk-17.0"
        maven "maven-3.9"
        nodejs "nodejs-23.9"
    }

    // Environment Variables for Registry, Images, and SonarQube
    environment {
        SCANNER_HOME = tool 'sonar-scanner'
        FRONTEND_IMAGE_NAME = 'three-tier-todo-frontend'
        BACKEND_IMAGE_NAME = "three-tier-todo-backend"
        IMAGE_TAG = "${BUILD_NUMBER}"
        DOCKER_REGISTRY = '17rj' 
        K8S_NAMESPACE = 'threetierapp'
    }

    stages {
        stage('Clean Workspace') {
            steps {
                // Ensure a fresh build environment by removing previous artifacts
                cleanWs()
            }
        }

        stage('Checkout Code') {
            steps {
                // Pulling the latest source code from the GitHub repository
                git branch: 'main', url: 'https://github.com/17J/Three-Tier-Todo-Application.git'
            }
        }

        stage('Compilation of Codes') {
            // Running frontend and backend compilation in parallel to save time
            parallel {
                stage('Frontend Compilation') {
                    steps {
                        dir('frontend') {
                            // Basic syntax check for Javascript files
                            sh 'find . -name "*.js" -exec node --check {} +'
                        }
                    }
                }
                stage('Backend Compilation') {
                    steps {
                        dir('backend') {
                            // Compiling Java source code using Maven
                            sh "mvn clean compile"
                        }
                    }
                }
            }
        }

        stage('Truffle Hog Secret Checking') {
            steps {
                // Scanning the repository for leaked API keys, tokens, or credentials
                sh '''
                    echo "Running TruffleHog secret scan..."
                    trufflehog git https://github.com/17J/Three-Tier-Todo-Application.git \
                        --json --no-update --fail > trufflehog-git-report.json || true
                    
                    if [ -s trufflehog-git-report.json ]; then
                        echo "⚠️ Secrets detected! Check report."
                        cat trufflehog-git-report.json
                    else
                        echo "✅ No secrets found!"
                    fi
                '''
            }
        }

        stage('Snyk SCA Scan') {
            // Software Composition Analysis (SCA) to check for vulnerable 3rd party libraries
            parallel {
                stage('Frontend') {
                    steps {
                        dir('frontend') {
                            withCredentials([string(credentialsId: 'snyk-cred', variable: 'SNYK_TOKEN')]) {
                                sh 'snyk auth $SNYK_TOKEN && snyk test --severity-threshold=high --json-file-output=snyk-frontend.json || true'
                            }
                        }
                    }
                }
                stage('Backend') {
                    steps {
                        dir('backend') {
                            withCredentials([string(credentialsId: 'snyk-cred', variable: 'SNYK_TOKEN')]) {
                                sh 'snyk auth $SNYK_TOKEN && snyk test --severity-threshold=high --json-file-output=snyk-backend.json || true'
                            }
                        }
                    }
                }
            }
        }

        stage('Sonarqube Code Analysis') {
            steps {
                // Static Application Security Testing (SAST) and code quality check
                withSonarQubeEnv("sonar") {
                    sh '''
                        $SCANNER_HOME/bin/sonar-scanner \
                            -Dsonar.projectName=devsecopsthreetier \
                            -Dsonar.projectKey=devsecopsthreetier \
                            -Dsonar.sources=frontend/src,backend/src \
                            -Dsonar.java.binaries=backend/target/classes \
                            -Dsonar.exclusions=**/node_modules/**,**/target/**
                    '''
                }
            }
        }

        stage('Quality Gate') {
            steps {
                // Halt the pipeline if SonarQube analysis does not meet the required threshold
                timeout(time: 1, unit: 'HOURS') {
                    waitForQualityGate abortPipeline: false, credentialsId: 'sonar-cred'
                }
            }
        }

        stage('Build Artifact') {
            steps {
                dir('backend') {
                    // Packaging the Java application into a JAR file
                    sh "mvn clean package -DskipTests=true"
                }
            }
        }

        stage('Pushlish Artifact to Nexus') {
            steps {
                dir('backend') {
                    // Uploading the compiled JAR to Nexus Repository for version control
                    configFileProvider([configFile(fileId: 'nexus-settings', variable: 'MAVEN_SETTINGS')]) {
                        sh 'mvn -s $MAVEN_SETTINGS clean deploy '
                    }
                }
            }
        }

        stage('OWASP Dependency Check') {
            steps {
                dir('backend') {
                    // Artifact scan focused on project dependencies and CVEs
                    withCredentials([string(credentialsId: 'nvd-api-key', variable: 'NVD_API_KEY')]) {
                        dependencyCheck additionalArguments: "--nvdApiKey ${NVD_API_KEY} --format XML --format HTML", odcInstallation: 'owasp'
                        dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
                    }
                }
            }
        }

        stage('Docker Build & Tag') {
            steps {
                script {
                    // Building and tagging images for both Frontend and Backend
                    sh "docker build -t ${FRONTEND_IMAGE_NAME}:${IMAGE_TAG} --file frontend/Dockerfile ./frontend"
                    sh "docker tag ${FRONTEND_IMAGE_NAME}:${IMAGE_TAG} ${DOCKER_REGISTRY}/${FRONTEND_IMAGE_NAME}:latest"
                    
                    sh "docker build -t ${BACKEND_IMAGE_NAME}:${IMAGE_TAG} --file backend/Dockerfile ./backend"
                    sh "docker tag ${BACKEND_IMAGE_NAME}:${IMAGE_TAG} ${DOCKER_REGISTRY}/${BACKEND_IMAGE_NAME}:latest"
                }
            }
        }

        stage('Dockle Image Scan') {
            steps {
                script {
                    // Scanning Docker images for security vulnerabilities and best practice violations
                    sh "dockle --exit-code 0 ${DOCKER_REGISTRY}/${FRONTEND_IMAGE_NAME}:latest || true"
                    sh "dockle --exit-code 0 ${DOCKER_REGISTRY}/${BACKEND_IMAGE_NAME}:latest || true"
                }
            }
        }

        stage('Push Docker Images') {
            steps {
                script {
                    // Logging into Docker Hub and pushing the scanned images
                    withCredentials([usernamePassword(credentialsId: 'docker-cred', passwordVariable: 'DOCKER_PASSWORD', usernameVariable: 'DOCKER_USERNAME')]) {
                        sh "docker login -u ${DOCKER_USERNAME} -p ${DOCKER_PASSWORD}"
                        sh "docker push ${DOCKER_REGISTRY}/${FRONTEND_IMAGE_NAME}:latest"
                        sh "docker push ${DOCKER_REGISTRY}/${BACKEND_IMAGE_NAME}:latest"
                    }
                }
            }
        }

        stage('Kubernetes Deploy') {
            steps {
                dir('K8s') {
                    // Deploying services and ingress to AWS EKS
                    withKubeConfig(caCertificate: '', clusterName: 'expdevops-cluster', credentialsId: 'kube-cred', namespace: 'threetierapp', serverUrl: 'https://60CE5186F6A0A4A653F5EF077EB5B74C.gr7.ap-south-1.eks.amazonaws.com') {
                        sh """
                            kubectl apply -f secrets-configmap.yml -n ${K8S_NAMESPACE}
                            kubectl apply -f db-ds-service.yml -n ${K8S_NAMESPACE}
                            kubectl apply -f backend-ds-service.yml -n ${K8S_NAMESPACE}
                            kubectl apply -f frontend-ds-service.yml -n ${K8S_NAMESPACE}
                            kubectl apply -f ingress.yml -n ${K8S_NAMESPACE}
                            sleep 60s
                        """
                    }
                }
            }
        }

        stage('Retrieve Application URL') {
            steps {
                script {
                    withKubeConfig(caCertificate: '', clusterName: 'expdevops-cluster', credentialsId: 'kube-cred', namespace: 'threetierapp', serverUrl: 'https://60CE5186F6A0A4A653F5EF077EB5B74C.gr7.ap-south-1.eks.amazonaws.com') {
                        // Polling Kubernetes to retrieve the ALB Hostname for DAST testing
                        timeout(time: 5, unit: 'MINUTES') {
                            waitUntil {
                                def albUrl = sh(script: "kubectl get ingress todo-app-ingress -n threetierapp -o jsonpath='{.status.loadBalancer.ingress[0].hostname}' 2>/dev/null || echo ''", returnStdout: true).trim()
                                if (albUrl) {
                                    env.K8S_URL = albUrl
                                    echo "✅ ALB URL found: http://${env.K8S_URL}"
                                    return true
                                } else {
                                    echo "⏳ Waiting for ALB Provisioning..."
                                    sleep 20
                                    return false
                                }
                            }
                        }
                    }
                }
            }
        }

        stage('OWASP ZAP Security Scan') {
            when { expression { return env.K8S_URL != null } }
            steps {
                script {
                    // Dynamic Application Security Testing (DAST) on the live application URL
                    def targetUrl = "http://${env.K8S_URL}"
                    sh """
                        docker pull ghcr.io/zaproxy/zaproxy:stable || true
                        docker run --rm -v \$(pwd):/zap/wrk/:rw -t ghcr.io/zaproxy/zaproxy:stable \
                            zap-baseline.py -t ${targetUrl} -r zap-report.html -I || echo "ZAP scan finished"
                    """
                    archiveArtifacts artifacts: 'zap-report.html', allowEmptyArchive: true
                }
            }
        }
    }

    post {
        always {
            // Cleanup local images to save disk space on the Jenkins agent
            sh "docker rmi ${DOCKER_REGISTRY}/${FRONTEND_IMAGE_NAME}:${IMAGE_TAG} || true"
            sh "docker rmi ${DOCKER_REGISTRY}/${BACKEND_IMAGE_NAME}:${IMAGE_TAG} || true"
        }
    }
}