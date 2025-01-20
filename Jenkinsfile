pipeline {
    agent any

    environment {
        // DockerHub credentials stored in Jenkins Credentials
        DOCKER_IMAGE = 'ravindradhakal/deadlocktest'
        DOCKER_REGISTRY = 'https://index.docker.io/v1/'
        DOCKER_USER = 'ravindradhakal' // Replace with your Docker Hub username
        DOCKER_CREDENTIALS = '2428d338-7ef1-47c4-8869-327b95a3d9eb'
        GITHUB_CREDENTIALS = '32d6cd06-b9ab-4237-89b9-f2728bcc5a98'

        KUBE_CONFIG_PATH = '/tmp/kubeconfig.yaml'  // Temporary path for kubeconfig
        GITHUB_REPO = 'https://github.com/ravindradhakal7/deadlocktest-k8s.git'
        GITHUB_BRANCH = 'main'  // Specify your branch

        EKS_CLUSTER_NAME = 'beautiful-alternative-sheepdog'  // EKS cluster name
        EKS_REGION = 'us-east-1'  // EKS region
        AWS_CREDENTIALS = '3086e787-624b-45ba-9d7e-13b3a57c987e'
        POM_VERSION = sh(script: "/opt/homebrew/bin/mvn help:evaluate -Dexpression=project.version -q -DforceStdout", returnStdout: true).trim()
    }

    stages {
        stage('Checkout Code') {
            steps {
                echo 'Fetching code from GitHub repository.'
                checkout([
                    $class: 'GitSCM',
                    branches: [[name: '*/main']], // Replace 'main' with your branch name
                    doGenerateSubmoduleConfigurations: false,
                    extensions: [],
                    userRemoteConfigs: [[
                        url: 'https://github.com/ravindradhakal7/deadlocktest.git',
                        credentialsId: '32d6cd06-b9ab-4237-89b9-f2728bcc5a98' // Use the ID of the credential you added
                    ]]
                ])
                echo 'Code checkout complete.'
            }
        }

        stage('Build') {
            steps {
                script {
                    // Extract POM version dynamically
                    def version = sh(script: "/opt/homebrew/bin/mvn help:evaluate -Dexpression=project.version -q -DforceStdout", returnStdout: true).trim()
                     echo "Version: ${version}"
                    // Print the extracted version
                    echo "POM Version: ${POM_VERSION}"
                    
                    // Build Docker image with the dynamic version
                    echo "Building Docker image with version: ${POM_VERSION}"
                    sh "docker build --build-arg VERSION=${POM_VERSION} -t ${DOCKER_IMAGE}:${POM_VERSION} ."
                }
            }
        }

        stage('Login to DockerHub') {
            steps {
                script {
                    withCredentials([string(credentialsId: DOCKER_CREDENTIALS, variable: 'DOCKER_TOKEN')]) {
                        sh '''
                            echo "$DOCKER_TOKEN" | docker login -u "$DOCKER_USER" --password-stdin $DOCKER_REGISTRY
                        '''
                        echo 'Successfully logged into DockerHub with token'
                    }
                }
            }
        }

        stage('Push Docker Image') {
            steps {
                script {
                    withCredentials([string(credentialsId: DOCKER_CREDENTIALS, variable: 'DOCKER_TOKEN')]) {
                        sh '''
                            echo "$DOCKER_TOKEN" | docker login -u "$DOCKER_USER" --password-stdin $DOCKER_REGISTRY
                        '''
                        // Push the image to Docker Hub with the version tag
                        sh "docker image ${DOCKER_IMAGE}:${POM_VERSION} push"
                        // Tag and push the "latest" version
                        sh "docker image ${DOCKER_IMAGE}:${POM_VERSION} tag ${DOCKER_IMAGE}:latest"
                        sh "docker image ${DOCKER_IMAGE}:latest push"
                        echo 'Docker image pushed successfully to DockerHub'
                    }
                }
            }
        }

        stage('Cleanup') {
            steps {
                script {
                    // Remove the local Docker image to free up space
                    sh "docker rmi ${DOCKER_IMAGE}:${POM_VERSION}"
                }
            }
        }
        
        stage('Clone GitHub Repository') {
            steps {
                git credentialsId: GITHUB_CREDENTIALS, url: "${GITHUB_REPO}", branch: "${GITHUB_BRANCH}"
            }
        }

        stage('Configure kubectl') {
            steps {
                script {
                    withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: env.AWS_CREDENTIALS]]) {
                        sh '''
                            aws eks --region ${EKS_REGION} update-kubeconfig --name ${EKS_CLUSTER_NAME}
                        '''
                    }
                }
            }
        }

        stage('Deploy to EKS') {
            steps {
                script {
                    sh "kubectl apply -f deployment.yaml"
                }
            }
        }
    }

    post {
        always {
            echo 'Cleaning up after build'
        }

        success {
            echo 'Build and Docker image push successful!'
        }

        failure {
            echo 'Build or push failed.'
        }
    }
}