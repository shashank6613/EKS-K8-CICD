pipeline {
    agent any

    parameters {
        string(name: 'DEPLOYMENT_NAME', defaultValue: 'my-deployment', description: 'Name of K8s Deployment')
        string(name: 'SERVICE_NAME', defaultValue: 'my-service', description: 'Name of K8s Service')
        string(name: 'CLUSTER_NAME', defaultValue: 'my-cluster', description: 'Name of the EKS cluster.')
        string(name: 'AWS_REGION', defaultValue: 'us-east-1', description: 'AWS region for the EKS cluster.')
        booleanParam(name: 'DELETE_CLUSTER_ON_FAILURE', defaultValue: false, description: 'Delete cluster if build fails.')
    }

    environment {
        CLUSTER_CREATED = 'false'
        AWS_CREDENTIALS_ID = 'aws-creds' 
        DOCKER_IMAGE_NAME = 'shashank9928/prt-app'
        DOCKER_CREDENTIAL_ID = 'dock-creds'
        GIT_CREDENTIALS_ID = 'git-creds'
        KUBE_DEPLOYMENT_FILE = 'deployment.yaml'
        KUBE_SERVICE_FILE = 'service.yaml'
    }

    stages {
        stage('Check/Create Cluster') {
            steps {
                script {
                    try {
                        withAWS(credentials: "${AWS_CREDENTIALS_ID}", region: "${params.AWS_REGION}") {
                            def clusterExists = sh(script: """
                                eksctl get cluster --region ${params.AWS_REGION} --name ${params.CLUSTER_NAME} --output json
                            """, returnStatus: true, shell: '/bin/bash')

                            if (clusterExists != 0) {
                                echo 'Cluster does not exist. Creating cluster...'
                                sh """
                                    eksctl create cluster \
                                        --name ${params.CLUSTER_NAME} \
                                        --version 1.30 \
                                        --region ${params.AWS_REGION} \
                                        --nodegroup-name Our-Node \
                                        --node-type t3.medium \
                                        --nodes 1 \
                                        --nodes-min 1 \
                                        --nodes-max 1 \
                                        --managed
                                """
                                echo "Cluster created successfully!"
                                currentBuild.description = 'Cluster Created'
                            } else {
                                echo 'Cluster already exists. Skipping creation.'
                                currentBuild.description = 'Cluster Exists'
                            }
                        }
                    } catch (Exception e) {
                        echo "Cluster check/creation failed: ${e}"
                        currentBuild.result = 'FAILURE'
                        currentBuild.description = 'Cluster Creation Failed'
                    }
                    // Debugging output to check the value of currentBuild.description
                    echo "Cluster Created Status: ${currentBuild.description}"
                }
            }
        }

        stage('Checkout Source Code') {
            when {
                expression { return currentBuild.description == 'Cluster Created' || currentBuild.description == 'Cluster Exists' }
            }
            steps {
                echo "Cluster Created Status: ${currentBuild.description}"
                git(
                    url: 'https://github.com/shashank6613/Website-PRT-ORG.git',
                    branch: 'main',
                    credentialsId: "${GIT_CREDENTIALS_ID}"
                )
            }
        }

        stage('Build Docker Image') {
            when {
                expression { return currentBuild.description == 'Cluster Created' || currentBuild.description == 'Cluster Exists' }
            }
            steps {
                script {
                    try {
                    def buildTag = "${DOCKER_IMAGE_NAME}:${BUILD_ID}"  // Tagging with BUILD_ID
                    docker.build(buildTag)
                    env.DOCKER_IMAGE_CREATED = 'true'
                    } catch (Exception e) {
                        echo "Docker build failed: ${e}"
                        env.DOCKER_IMAGE_CREATED = 'false'  // Mark as false if build fails
                        currentBuild.result = 'FAILURE'     // Mark the build as failed
                    }
                }
            }
        }

        stage('Push Docker Image') {
            when {
                expression { return currentBuild.description == 'Cluster Created' || currentBuild.description == 'Cluster Exists' }
            }
            steps {
                script {
                    def buildTag = "${DOCKER_IMAGE_NAME}:${BUILD_ID}"  // Tagging with BUILD_ID
                    docker.withRegistry('https://index.docker.io/v1/', "${DOCKER_CREDENTIAL_ID}") {
                    docker.image(buildTag).push()
                    }
                }
            }
        }

        stage('Update kubeconfig') {
            when {
                expression { return currentBuild.description == 'Cluster Created' || currentBuild.description == 'Cluster Exists' }
            }
            steps {
                withAWS(credentials: "${AWS_CREDENTIALS_ID}", region: "${params.AWS_REGION}") {
                    script {
                // Update kubeconfig
                        sh """
                            aws eks --region ${params.AWS_REGION} update-kubeconfig --name ${params.CLUSTER_NAME}
                        """
                // Use the Docker image tagged with BUILD_ID
                        def dockerImage = "${DOCKER_IMAGE_NAME}:${BUILD_ID}"
                        sh """
                            sed -i 's|image: .*|image: ${dockerImage}|' ${KUBE_DEPLOYMENT_FILE}
                        """
                    }
                }
            }
        }

        stage('Deploy to EKS') {
            when {
                expression { return currentBuild.description == 'Cluster Created' || currentBuild.description == 'Cluster Exists' }
            }
            steps {
                script {
                    withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-creds']]) {
                        sh "kubectl apply -f ${KUBE_SERVICE_FILE}"
                        sh "kubectl apply -f ${KUBE_DEPLOYMENT_FILE}"
                    }    
                }
            }
        }
    }

    post {
        success {
            echo 'Pipeline completed successfully.'
        }
        failure {
            echo 'Pipeline failed. Cleaning up resources if needed.'

            script {
                // Clean up Docker resources if created
                if (env.DOCKER_IMAGE_CREATED == 'true') {
                    echo 'Cleaning up Docker containers and images.'
                    withAWS(credentials: "${AWS_CREDENTIALS_ID}", region: "${params.AWS_REGION}") {
                        script {
                            sh '''
                                docker ps -q --filter "ancestor=${DOCKER_IMAGE_NAME}:latest" | xargs -r docker stop
                                docker ps -a -q --filter "ancestor=${DOCKER_IMAGE_NAME}:latest" | xargs -r docker rm
                                docker rmi ${DOCKER_IMAGE_NAME}:latest || true
                            '''
                        }
                    }
                }

                // Clean up EKS resources (deployment and service)
                    echo 'Cleaning up EKS resources if created.'
                    withAWS(credentials: "${params.AWS_CREDENTIALS_ID}", region: "${params.AWS_REGION}") {
                        script {
                            sh '''
                                kubectl delete deployment ${params.DEPLOYMENT_NAME} --ignore-not-found
                                kubectl delete service ${params.SERVICE_NAME} --ignore-not-found
                            '''
                        }
                    }
                }
            }
        }    
    }
}
