pipeline {
    agent any
        environment {
        // Set environment variables for Docker registry, image name, etc.
            DOCKER_IMAGE = 'prt-app'  // Change to your Docker image name
            DOCKER_REGISTRY = 'docker.io'  // Replace with your Docker registry
            DOCKER_REPO = 'shashank9928/prt-app'
            KUBECONFIG = 'local-kubeconfig'
        }
        stages {
            stage('Checkout') {
                steps {
                    checkout scm
                }
            }

            stage('Build Docker Image') {
                steps {
                    script {
                        def buildTag = "${env.BUILD_ID}"
                    
                        def image = docker.build("${DOCKER_REGISTRY}/${DOCKER_REPO}:${buildTag}")
                    }
                }
            }
        
            stage('Push Docker Image') {
                steps {
                    script {
                        withCredentials([usernamePassword(credentialsId: 'dock-creds', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                        
                            sh "echo $DOCKER_PASS | docker login --username $DOCKER_USER --password-stdin"
                            def buildTag = "${env.BUILD_ID}"
                            def imageName = "${DOCKER_REGISTRY}/${DOCKER_REPO}:${buildTag}"
                            sh "docker push ${imageName}"
                        }
                    }
                }
            }

            stage('Deploy to Local Kubernetes') {
                agent { label 'k8-master' }
                steps {
                    withCredentials([file(credentialsId: 'local-kubeconfig', variable: 'KUBECONFIG')]) {
                        script {
                            def buildTag = "${env.BUILD_ID}"
                            sh "kubectl apply -f deployment.yaml"

                            sh "kubectl set image deployment/my-app-deployment prt-app=${DOCKER_REGISTRY}/${DOCKER_REPO}:${buildTag}"
                
                            sh "kubectl apply -f service.yaml"
                
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
            
            script {
                
                sh "docker ps -q --filter \"ancestor=${DOCKER_REGISTRY}/${DOCKER_REPO}:${env.BUILD_ID}\" | xargs -r docker stop"

                sh "docker ps -a -q --filter \"ancestor=${DOCKER_REGISTRY}/${DOCKER_REPO}:${env.BUILD_ID}\" | xargs -r docker rm"

                sh "docker images -q ${DOCKER_REGISTRY}/${DOCKER_REPO}:${env.BUILD_ID} | xargs -r docker rmi"
            }
        }
    }
}
