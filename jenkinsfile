pipeline {
    agent any

    environment {
        DOCKER_REGISTRY = "docker.io"
        DOCKER_IMAGE = "shashank6613/prt-image"
        K8S_DEPLOYMENT_FILE = "deployment.yaml"
        K8S_SERVICE_FILE = "service.yaml"
        DOCKERHUB_CREDENTIALS = "dock-creds" // Set your Jenkins DockerHub credentials ID
        K8S_CREDENTIALS = "k8-creds" // Set your Jenkins Kubernetes credentials ID
        IMAGE_TAG = "${env.BUILD_ID}"
    }

    stages {
        stage('Checkout SCM') {
            steps {
                checkout scm
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    // Build Docker image
                    sh "docker build -t ${DOCKER_IMAGE}:${env.BUILD_ID} ."
                }
            }
        }

        stage('Login to DockerHub') {
            steps {
                script {
                    // Login to DockerHub
                    withCredentials([usernamePassword(credentialsId: DOCKERHUB_CREDENTIALS, usernameVariable: 'DOCKER_USERNAME', passwordVariable: 'DOCKER_PASSWORD')]) {
                        sh "echo ${DOCKER_PASSWORD} | docker login -u ${DOCKER_USERNAME} --password-stdin"
                    }
                }
            }
        }

        stage('Push Docker Image to DockerHub') {
            steps {
                script {
                    // Push Docker image to DockerHub
                    sh "docker push ${DOCKER_IMAGE}:${env.BUILD_ID}"
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                script {
                    // Deploy using kubectl
                    withCredentials([file(credentialsId: K8S_CREDENTIALS, variable: 'KUBE_CONFIG')]) {
                        // Update the Kubernetes deployment and service files
                        def buildId = env.BUILD_ID
                        sh """
                            kubectl --kubeconfig=\$KUBE_CONFIG set image deployment/prt-app prt-app=${DOCKER_IMAGE}:${buildId} --record
                            kubectl --kubeconfig=\$KUBE_CONFIG apply -f ${K8S_DEPLOYMENT_FILE}
                            kubectl --kubeconfig=\$KUBE_CONFIG apply -f ${K8S_SERVICE_FILE}
                        """
                    }
                }
            }
        }
    }

    post {
        success {
            echo "Deployment completed successfully!"
        }

        failure {
            echo "Build or deployment failed. Please check the logs."
        }
    }
}
