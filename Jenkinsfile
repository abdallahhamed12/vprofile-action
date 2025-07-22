pipeline {
    agent any
    tools {
        maven "MAVEN3.9" 
    }
    
    environment {
        DOCKER_REGISTRY = "abdallahhamed" // e.g., "docker.io" or your private registry URL
        DOCKER_CREDENTIAL_ID = "docker-hub-credentials" // Jenkins credentials ID for Docker registry
        DOCKER_IMAGE_NAME = "vprofile-app"
        DOCKER_IMAGE_TAG = "${env.BUILD_ID}"
        KUBECONFIG = "/root/.kube/config" // Path to kubeconfig file in Jenkins
        K8S_NAMESPACE = "vprofile-namespace"
        APP_NAME = "vprofile-app"
        APP_PORT = 8080 // Your application port
    }
    
    stages {
        stage('BUILD') {
            steps {
                sh 'mvn clean install -DskipTests'
            }
            post {
                success {
                    echo 'Now Archiving...'
                    archiveArtifacts artifacts: '**/target/*.war'
                }
            }
        }

        stage('UNIT TEST') {
            steps {
                sh 'mvn test'
            }
        }

        stage('INTEGRATION TEST') {
            steps {
                sh 'mvn verify -DskipUnitTests'
            }
        }
        
        stage('CODE ANALYSIS WITH CHECKSTYLE') {
            steps {
                sh 'mvn checkstyle:checkstyle'
            }
            post {
                success {
                    echo 'Generated Analysis Result'
                }
            }
        }
        
        stage('Build and Push Docker Image') {
            steps {
                script {
                    // Build Docker image
                    docker.build("${DOCKER_REGISTRY}/${DOCKER_IMAGE_NAME}:${DOCKER_IMAGE_TAG}", "--build-arg WAR_FILE=target/*.war .")
                    
                    // Authenticate with Docker registry
                    docker.withRegistry("https://${DOCKER_REGISTRY}", DOCKER_CREDENTIAL_ID) {
                        // Push Docker image
                        docker.image("${DOCKER_REGISTRY}/${DOCKER_IMAGE_NAME}:${DOCKER_IMAGE_TAG}").push()
                        
                        // Optionally push as latest
                        docker.image("${DOCKER_REGISTRY}/${DOCKER_IMAGE_NAME}:${DOCKER_IMAGE_TAG}").push('latest')
                    }
                }
            }
        }
        
        stage('Deploy to Kubernetes') {
            steps {
                script {
                    // Check if deployment already exists
                    def deploymentExists = sh(
                        script: "kubectl get deployment ${APP_NAME} -n ${K8S_NAMESPACE} --ignore-not-found",
                        returnStatus: true
                    ) == 0
                    
                    if (!deploymentExists) {
                        // Create namespace if it doesn't exist
                        sh "kubectl create namespace ${K8S_NAMESPACE} --dry-run=client -o yaml | kubectl apply -f -"
                        
                        // Create deployment
                        sh """
                            cat <<EOF | kubectl apply -f -
                            apiVersion: apps/v1
                            kind: Deployment
                            metadata:
                              name: ${APP_NAME}
                              namespace: ${K8S_NAMESPACE}
                              labels:
                                app: ${APP_NAME}
                            spec:
                              replicas: 2
                              selector:
                                matchLabels:
                                  app: ${APP_NAME}
                              template:
                                metadata:
                                  labels:
                                    app: ${APP_NAME}
                                spec:
                                  containers:
                                  - name: ${APP_NAME}
                                    image: ${DOCKER_REGISTRY}/${DOCKER_IMAGE_NAME}:${DOCKER_IMAGE_TAG}
                                    ports:
                                    - containerPort: ${APP_PORT}
                                    resources:
                                      requests:
                                        cpu: "100m"
                                        memory: "256Mi"
                                      limits:
                                        cpu: "500m"
                                        memory: "512Mi"
                            EOF
                        """
                        
                        // Create service
                        sh """
                            cat <<EOF | kubectl apply -f -
                            apiVersion: v1
                            kind: Service
                            metadata:
                              name: ${APP_NAME}-svc
                              namespace: ${K8S_NAMESPACE}
                              labels:
                                app: ${APP_NAME}
                            spec:
                              type: NodePort
                              selector:
                                app: ${APP_NAME}
                              ports:
                              - protocol: TCP
                                port: 80
                                targetPort: ${APP_PORT}
                            EOF
                        """
                        
                        // Wait for deployment to be ready
                        sh "kubectl rollout status deployment/${APP_NAME} -n ${K8S_NAMESPACE} --timeout=300s"
                    } else {
                        echo "Deployment already exists. Use a different pipeline for updates."
                        // Alternatively, you could update the existing deployment here
                        // sh "kubectl set image deployment/${APP_NAME} ${APP_NAME}=${DOCKER_REGISTRY}/${DOCKER_IMAGE_NAME}:${DOCKER_IMAGE_TAG} -n ${K8S_NAMESPACE}"
                    }
                }
            }
        }
    }
}
