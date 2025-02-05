pipeline {
    agent any
    environment {
        // Set environment variables for Docker registry, image name, etc.
        DOCKER_IMAGE = 'prt-app'  // Change to your Docker image name
        DOCKER_REGISTRY = 'docker.io'  // Replace with your Docker registry
        DOCKER_REPO = 'shashank9928/prt-app'
    }
    stages {
        stage('Checkout') {
            steps {
                // Checkout the code from your Git repository
                checkout scm
            }
        }
        
        stage('Build Docker Image') {
            steps {
                script {
                    // Use Jenkins' build ID to tag the Docker image
                    def buildTag = "${env.BUILD_ID}"
                    // Build the Docker image and tag it with the build ID
                    sh "docker build -t ${DOCKER_REGISTRY}/${DOCKER_REPO}:${buildTag} ."
                }
            }
        }
        
        stage('Push Docker Image') {
            steps {
                script {
                    // Use credentials to log in to Docker Hub
                    withCredentials([usernamePassword(credentialsId: 'dock-creds', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                        // Log in to Docker Hub before pushing the image
                        sh "echo $DOCKER_PASS | docker login --username $DOCKER_USER --password-stdin"

                        // Push the Docker image to the registry with the build ID tag
                        def buildTag = "${env.BUILD_ID}"
                        sh "docker push ${DOCKER_REGISTRY}/${DOCKER_REPO}:${buildTag}"
                    }
                }
            }
        }
        
        stage('Deploy to Local Kubernetes') {
            steps {
                        // Using withCredentials block to securely bind the kubeconfig file (if required)
                withCredentials([file(credentialsId: 'local-kubeconfig', variable: 'KUBECONFIG')]) {
                    script {
                        // Set up the Docker image with the build ID tag for deployment
                        def buildTag = "${env.BUILD_ID}"
                
                        // Apply the Kubernetes deployment (make sure you have the correct YAML files)
                        sh "kubectl apply -f deployment.yaml"
                
                        // Optionally, you can update the image of your deployment using kubectl
                        // Update the deployment with the latest image tag
                        sh "kubectl set image deployment/my-app-deployment prt-app=${DOCKER_REGISTRY}/${DOCKER_REPO}:${buildTag}"
                
                        // Apply the service if necessary
                        sh "kubectl apply -f service.yaml"
                
                        // Optional: You can check if the deployment is successful
                        sh "kubectl rollout status deployment/my-app-deployment"
                    }
                }
            }
        }
    }
    post {
        success {
            echo "Build and deployment successful"
        }
        failure {
            echo "Build or deployment failed. Cleaning up Docker images and containers."
            
            // Clean up Docker images and containers left running after a failed deployment
            script {
                // Find and stop any running containers (replace 'my-app' with your container name if needed)
                sh 'docker ps -q --filter "ancestor=${DOCKER_REGISTRY}/${DOCKER_REPO}:${env.BUILD_ID}" | xargs -r docker stop'

                // Remove any containers that were running from this build
                sh 'docker ps -a -q --filter "ancestor=${DOCKER_REGISTRY}/${DOCKER_REPO}:${env.BUILD_ID}" | xargs -r docker rm'

                // Remove the Docker image if it exists
                sh 'docker images -q ${DOCKER_REGISTRY}/${DOCKER_REPO}:${env.BUILD_ID} | xargs -r docker rmi'
            }
        }
    }
}

