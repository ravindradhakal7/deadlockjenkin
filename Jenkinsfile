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
        stage('Extract Version') {
            steps {
                script {
                    // Extract the version from pom.xml using Maven
                    def version = sh(script: "mvn help:evaluate -Dexpression=project.version -q -DforceStdout", returnStdout: true).trim()
                    env.VERSION = version
                    echo "Extracted Version: ${env.VERSION}"
                }
            }
        }

        stage('Build') {
            steps {
                // Build the Docker image
                script {
                    docker.build("${DOCKER_IMAGE}:${VERSION}")
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
                    // Use a "Secret Text" credential for Docker Hub token
                    withCredentials([string(credentialsId: DOCKER_CREDENTIALS, variable: 'DOCKER_TOKEN')]) {
                        sh '''
                            echo "$DOCKER_TOKEN" | docker login -u "$DOCKER_USER" --password-stdin $DOCKER_REGISTRY
                        '''
                        docker.image("${DOCKER_IMAGE}:${VERSION}").push()
                        docker.image("${DOCKER_IMAGE}:${VERSION}").tag('latest')
                        docker.image("${DOCKER_IMAGE}:latest").push()
                        echo 'Docker image pushed successfully to DockerHub'
                    }
                }
            }
        }

        stage('Cleanup') {
            steps {
                // Remove the local Docker image to free up space
                script {
                    sh "docker rmi ${DOCKER_IMAGE}:${VERSION}"
                }
            }
        }
        
        stage('Clone GitHub Repository') {
            steps {
                // Clone the repository containing the deployment.yaml file
                // git url: "${GITHUB_REPO}", branch: "${GITHUB_BRANCH}"
                git credentialsId: GITHUB_CREDENTIALS, url: "${GITHUB_REPO}", branch: "${GITHUB_BRANCH}"
            }
        }

        stage('Configure kubectl') {
            steps {
                 script {
                        // Use AmazonWebServicesCredentialsBinding to securely inject AWS credentials
                        withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: env.AWS_CREDENTIALS]]) {
                            // Now that AWS credentials are available, run the AWS CLI command
                            sh '''
                                # Configure AWS CLI with the access keys
                                aws eks --region ${EKS_REGION} update-kubeconfig --name ${EKS_CLUSTER_NAME}
                            '''
                        }
                 }
            }
        }

        stage('Deploy to EKS') {
            steps {
                script {
                    // Apply the deployment.yaml file from the cloned repository
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
