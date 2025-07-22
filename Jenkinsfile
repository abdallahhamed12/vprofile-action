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
    } // This closes the stages block
} // This closes the pipeline block
